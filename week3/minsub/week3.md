## 3주차 스터디 개요
> RHEL에서 디스크와 파티션, 파일 시스템, 마운트 구조를 이해하고, 저장 공간을 유연하게 관리하기 위한 LVM 개념과 기본 실습을 진행하고자 한다.

1주차에서는 디렉터리 구조와 기본 명령어, 2주차에서는 패키지 및 사용자/권한 관리 방법을 학습했다.
이번 3주차에서는 한 단계 나아가 `스토리지 관점에서 RHEL을 다루는 기본 방법` 스터디를 목표로 한다.

---

## 1. 기본 개념 (디스크, 파티션, 볼륨, 파일 시스템, 마운트)

### 1-1. 디스크
디스크는 실제 데이터를 저장하는 장치 자체를 의미한다.

가상 머신 환경에서는 VMware에 추가한 가상 디스크도 리눅스 입장에서는 하나의 디스크로 인식된다.

예를 들어
(디스크가 추가될 때마다 문자가 알파벳순으로 증가한다.)

- /dev/sda → 첫 번째 SCSI/SATA 계열 디스크
- /dev/sdb → 두 번째 SCSI/SATA 계열 디스크
- /dev/nvme0n1 → 첫 번째 NVMe 디스크

리눅스에서는 디스크 장치 전체를 `/dev` 아래의 장치 파일로 표현한다.
예를 들어 `/dev/sda`, `/dev/sdb`, `/dev/nvme0n1`은 디스크 전체를 의미한다.

반면 `/dev/sda1`, `/dev/sdb1`, `/dev/nvme0n1p1` 처럼 뒤에 파티션 번호가 붙은 형태는 해당 디스크를 나눈 파티션을 의미한다.

즉, 디스크는 저장 공간의 가장 큰 단위라고 볼 수 있다.

### 1-2. 파티션
하나의 물리적인 디스크를 여러 개의 논리적인 디스크로 나누는 것.

하나의 디스크를 하나의 통째 저장공간으로 사용할 수도 있지만, 필요에 따라 여러 구역으로 나누어 각각 다른 용도로 사용할 수 있다.

예를 들어

- /dev/sdb1
- /dev/dev2

파티션 번호를 붙여, 해당 디스크를 나눈 파티션이 된다.

### 리눅스 파티션 특징

리눅스 파티션은 주(Primary) 파티션과 확장(Extended) 파티션, 논리(Logical) 파티션으로 구성할 수 있다.

- 확장 파티션은 데이터 저장 용도보다는 실제 데이터가 저장되는 논리 파티션을 만들기 위한 툴이다.
    - 파일 시스템 생성, 마운트 불가
- 디스크 하나당 주 파티션을 최대 4개까지 생성할 수 있다.
- 4개 이상의 파티션이 필요한 경우, 확장과 논리 파티션이 필요하며, 최대 12개까지 생성이 가능하다.

### 1-3. 볼륨
볼륨은 운영체제가 하나의 저장공간 단위로 인식하고 사용하는 논리적 저장 영역이다. 일반적인 파티션 방식에서는 파티션이 곧 사용 단위가 되지만, LVM 환경에서는 Logical Volume(LV)이 실제 사용 단위가 된다. 즉 볼륨은 파일 시스템을 생성하고 마운트하여 실제 저장공간처럼 사용하는 대상이라고 볼 수 있다.

- 일반 방식: 파티션이 곧 실사용 저장공간
- LVM 방식: LV가 곧 실사용 저장공간

볼륨은 디스크나 파티션 위에 만들어져서 운영체제가 실제 저장공간처럼 쓰는 단위이다.

조금 헷갈리는 부분이 볼륨은 문맥마다 의미가 넓게 쓰일 수 있다.

어떤 파티션 위에 파일 시스템을 만들면 그걸 그냥 볼륨이라고 부를 수 있다. 즉, 윈도우의 C: 드라이브 같은 것도 일종의 볼륨 개념이다.

그런데, 리눅스 환경에서는 보통 LVM 문맥에서의 볼륨을 말하는데, 이는 아래의 LVM 개념에서 보다 자세히 정리한다.

### 1-4. 파일 시스템
파일 시스템은 디스크나 파티션에 파일을 저장하고 관리할 수 있도록 구조를 잡아주는 방식이다. 파일 시스템이 생성되어야 운영체제가 해당 공간을 실제로 사용할 수 있다.

리눅스의 대표적인 파일시스템:

- `xfs` : 64-bit 고성능 저널링 파일 시스템
- `ext3` : 저널링 파일시스템, ext2보다 파일시스템의 복수/보안 기능이 크게 향상
- `ext4` : 16TB까지만 지원하던 ext3과는 달리 더 큰 용량을 지원, 삭제된 파일 복구, 속도가 더 빨라진 파일 시스템
- `ios9660` : DVD/CD-ROM을 위한 표준 파일 시스템으로 읽기만 가능
- `swap` : swap 공간으로 사용되는 파일 시스템

즉, 디스크와 파티션이 물리적/논리적 공간이라면, 파일 시스템은 그 공간을 실제 파일 저장소로 사용할 수 있게 해주는 구조라고 볼 수 있다.

### 1-5. 마운트
마운트는 파일 시스템을 특정 디렉터리에 연결하는 작업이다.

예를 들어 `/dev/sdb1`을 `/data`에 마운트하면, 사용자는 `/data` 디렉터리를 통해 해당 디스크 공간을 사용할 수 있다.

즉, 마운트는 저장 장치를 리눅스 디렉터리 구조 안으로 연결하는 과정이다.

참고자료: [디스크, 파티션과 볼륨의 차이](https://soft.plusblog.co.kr/46#google_vignette),
[Linux 파일 시스템](https://neul-carpediem.tistory.com/98#google_vignette)

---

## 2. LVM이란?
LVM(Logical Volume Manager)은 리눅스에서 저장공간을 보다 유연하게 관리하기 위한 구조이다.

일반 파티션 방식은 디스크를 고정적으로 나누기 때문에, 나중에 크기 변경이나 통합이 번거로울 수 있다. 반면 LVM은 여러 저장장치를 하나의 풀처럼 묶어서, 그 안에서 필요한 크기만큼 논리 볼륨을 생성할 수 있다.

LVM 구조는 아래와 같다.

- PV (Physical Volume): 실제 디스크 또는 파티션
- VG (Volume Group): PV를 묶어 만든 저장공간 그룹
- LV (Logical Volume): VG에서 필요한 만큼 떼어낸 논리 볼륨

즉, 디스크를 바로 사용하는 대신

`PV → VG → LV → 파일 시스템 → 마운트`

순서로 더 유연하게 저장공간을 관리할 수 있는 것이다.

### 2-1. 일반 파티션 방식 vs LVM 방식 비교
> 일반 파티션 방식
   `디스크 → 파티션 → 파일 시스템 → 마운트`

예를 들어

- `/dev/sdb`
- `/dev/sdb1`
- `mkfs.xfs /dev/sdb1`
- `mount /dev/sdb1 /data`

여기서는 /dev/sdb1/ 이 실질적인 사용 단위가 되고, 이걸 넓게 보면 볼륨이라고도 하지만, 보통 그냥 파티션이라고 칭한다.

> LVM 방식
  `디스크 → PV → VG → LV → 파일 시스템 → 마운트`

- `/dev/sdc`
- `pvcreate /dev/sdc`
- `vgcreate vgdata /dev/sdc`
- `lvcreate -L 3G -n lvdata vgdata`
- `mkfs.xfs /dev/vgdata/lvdata`
- `mount /dev/vgdata/lvdata /lvdata`

여기서는 `/dev/vgdata/lvdata`  가 볼륨이 된다.

---
## 실습

### 1. 현재 시스템 스토리 구조 확인
실습에 본격적으로 들어가기 전, 현재 RHEL이 어떤 디스크와 파일 시스템 구조를 가지고 있는지 먼저 확인

<p align="center">
<img width="45%" height="207" alt="image" src="https://github.com/user-attachments/assets/7abd5997-b430-4ee8-8c62-53dad2a6f764" />
<img width="45%" height="294" alt="image" src="https://github.com/user-attachments/assets/0de632b6-24ab-4656-9e72-2bc5945b4e9e" />
</p>

<img width="45%" height="405" alt="image" src="https://github.com/user-attachments/assets/efabe60e-ec9d-441e-97b5-2699575bd5e0" />



- `lsblk` : 디스크, 파티션, 마운트 구조 확인
- `df -h` : 마운트된 파일 시스템의 용량 및 사용량 확인
- `mount` : 현재 마운트된 파일 시스템 확인
- `blkid` : 디스크/파티션의 UUID 및 파일 시스템 타입 확인

이 명령어들을 통해 현재 OS가 어떤 디스크에 설치되어 있고, 새로 추가한 디스크가 어떤 이름으로 인식되는지 파악할 수 있다.

### 2. 새 디스크 추가 및 확인
기존 시스템 디스크를 실습 대상으로 사용하는 것은 위험할 수 있으므로, VMware에서 새 가상 디스크를 추가한 뒤 이를 기준으로 실습을 진행했다.

디스크 추가 후 다시 아래 명령어를 실행해 새 디스크가 정상적으로 인식되었는지 확인했다.

- `lsblk:` 블록 디바이스 구조 확인
- `sudo fdisk -l` : 전체 디스크 및 파티션 정보 출력

<img width="476" height="131" alt="image" src="https://github.com/user-attachments/assets/0e061a3e-3bf1-4bda-9fc4-a11423872fd4" />


### 3. 파티션 생설 실습
새 디스크를 바로 사용하는 대신, 먼저 파티션을 생성했다.

<img width="661" height="458" alt="image" src="https://github.com/user-attachments/assets/b4d48387-e127-4649-ab8c-fe1e645da04d" />


`fdisk`는 디스크의 파티션을 생성, 삭제, 수정할 수 있는 명령어이다.

아래의 명령 순서로 파티션을 생성했다.

`sudo fdisk /deb/sda`
`fdisk`는 디스크의 파티션을 생성, 삭제, 수정할 수 있는 명령어이다.

- `n` : 새 파티션 생성
- `p` : primary 파티션 선택
- 파티션 넘버 지정
- `엔터`: 기본 시작 섹터 사용
- `엔터`: 기본 끝 섹터 사용
- `w`: 저장 후 종료

<img width="655" height="144" alt="image" src="https://github.com/user-attachments/assets/6bb32db7-ad16-4be5-b668-02740ba85824" />

`sda1` 이라는 새 파티션이 생긴 것을 확인할 수 있다.

### 4. 파일 시스템 생성
파티션을 생성한 뒤에는, 해당 공간을 실제로 사용할 수 있도록 파일 시스템을 생성해야 한다.

실습에서는 RHEL에서 많이 사용되는 xfs 파일 시스템을 생성했다.

<img width="646" height="265" alt="image" src="https://github.com/user-attachments/assets/c1d93f22-0304-417d-8b1b-c51757df8fa8" />


`sudo mkfs.xfs /dev/sda1`

- `mkfs` : make filesystem의 약자
- `mkfs.xfs: xfs` : `xfs` 타입 파일 시스템 생성

파일 시스템 생성 후 아래 명령어로 파일 시스템 타입과 UUID를 확인했다.

<img width="653" height="65" alt="image" src="https://github.com/user-attachments/assets/e0fd93d6-7e22-4712-a137-629bbb423328" />


`sudo blkid /deb/sda1`
이 과정을 통해 `/dev/sda1` 이 `xfs` 파일 시스템으로 초기화되었음을 확인했다.

### 5. 마운트 실습
파일 시스템을 생성한 뒤, 이를 특정 디렉터리에 연결해야 실제 저장공간처럼 사용할 수 있다.
새 파티션을 `/data` 디렉터리에 마운트했다.

<p align="center">
<img width="45%" height="406" alt="image" src="https://github.com/user-attachments/assets/4c7bcfde-8502-4c70-889e-799900517919" />
<img width="45%" height="252" alt="image" src="https://github.com/user-attachments/assets/d4ee65e0-e8f0-4647-a4c5-0f087bfa091e" />
</p>

- `df -h` : `/data` 가 새로운 파일 시스템으로 연결되었는지 확인
- `mount | grep /data` :  `/data`  마운트 여부 확인
- `lsblk` : 디스크와 마운트 포인트 관계 재확인

<img width="426" height="145" alt="image" src="https://github.com/user-attachments/assets/f7aa3211-aa5e-4a12-b1b0-f5516921be54" />


마운트가 완료된 뒤, 디렉터리에 파일을 만들어보며 정상적으로 동작하는지 테스트

### 6. 자동 마운트 설정 (/etc/fstab)
직접 `mount` 명령으로 연결한 파일 시스템은 재부팅 시 자동으로 유지되지 않을 수 있다.
이를 해결하기 위해 `/etc/fstab` 파일에 자동 마운트 정보를 등록한다.

- UUID 확인

<img width="319" height="68" alt="image" src="https://github.com/user-attachments/assets/b212fadd-3e46-4ff2-9d3b-6edafeea65cc" />


- `/etc/fstab` 파일 편집 (`UUID = 장치 UUID /data xfs defaults 0 0`)
- `sudo mount -a`로 /etc/fstab 내용을 기준으로 전체 마운트 테스트

설정 정상 확인 완료

---

### 7. LVM 실습
이번에는 일반 파티션 방식이 아닌 LVM 구조를 사용해 저장공간을 생성하고자 한다.

우선 이를 위해 디스크를 하나 더 추가해주었다.

<img width="450" height="115" alt="image" src="https://github.com/user-attachments/assets/52c29965-7cc2-41d0-b32d-56d7d775c5cf" />


### 7-1. PV 생성

<img width="423" height="50" alt="image" src="https://github.com/user-attachments/assets/93a70e41-7886-493b-8084-407a71a5a1f1" />


- `vgcreate`: 디스크 또는 파티션을 LVM의 Physical Volume으로 초기화

### 7-2. VG 생성

<img width="427" height="42" alt="image" src="https://github.com/user-attachments/assets/6ed6826d-4602-4791-be39-c3cfde81a9c8" />


- `vgcreate`: PV를 묶어 Volume Group 생성
- `vgdata` : 생성할 VG 이름 지정

### 7-3. LV 생성

<img width="495" height="46" alt="image" src="https://github.com/user-attachments/assets/91381b45-1536-4f24-be5b-5ca4d875c7fd" />


- `lvcreate`: Logical Volume 생성
- `-L 3G` : 크기 3GB
- `-n lvdata` : LV 이름

### 7-4. 파일 시스템 생성
생성한 논리 볼륨에 `xfs` 파일 시스템 생성

<img width="646" height="250" alt="image" src="https://github.com/user-attachments/assets/3113e5a2-0db0-43c9-9181-4cfac5246e31" />


### 7-5. 마운트 포인트 생성 및 마운트
- `sudo mkdir /lvdata` : 마운트할 포인트 생성
- `sudo mount /dev/vgdata/lvdata /lvdata` : 마운트

마운트 한 뒤에 결과 확인

<p align="center">
<img width="45%" height="391" alt="image" src="https://github.com/user-attachments/assets/218b20bc-abe9-439d-93a6-724b9c8cccaf" />
<img width="45%" height="281" alt="image" src="https://github.com/user-attachments/assets/0c8a5380-29f6-42ca-b480-4e8eb22727f4" />
</p>

<img width="50%" height="347" alt="image" src="https://github.com/user-attachments/assets/dc4d224f-8716-4fe2-bb1b-a6f6a66cea4e" />


- `pvs` : Physical Volume 정보 확인
- `vgs` : Volume Group 정보 확인
- `lvs`: Logical Volume 정보 확인

이를 통해 LVM이 일반 파티션보다 더 유연한 구조로 저장공간을 관리한다는 점을 확인

## 정리
이번 3주차 스터디에서는 아래 내용을 익혔다.

- 디스크, 파티션, 파일 시스템, 마운트의 차이
- 새 디스크를 추가한 뒤 파티션 생성 방법
- xfs 파일 시스템 생성 방법
- 마운트를 통해 저장공간을 특정 디렉터리에 연결하는 법
- /etc/fstab 을 통해 자동 마운트 설정하는 법
- LVM의 기본 구조 (PV, VG, LV)와 논리 볼륨 생성 과정

> 특히, 리눅스에서 저장 장치는 단순히 연결하는 것으로 끝나는 것이 아니라, `디스크 인식 → 파티션 생성 → 파일 시스템 생성 → 마운트 → 자동 마운트 설정` 이라는 흐름을 거쳐야 실제 운영에 활용할 수 있다는 점을 이해했다. 또한, LVM을 통해 저장공간을 보다 유연하게 관리할 수 있다는 점도 확인하며, 이는 이후 서버 운영이나 인프라 관리 관점에서도 중요한 기초 개념이 될 것이라 생각한다.
