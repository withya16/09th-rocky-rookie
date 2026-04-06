## 1. 물리 계층 (Physical Layer): 비트의 저장과 정렬

데이터가 전기적 신호 혹은 자기적 배열로 저장되는 물리적 단계입니다. 커널은 이 물리적 특성을 추상화하여 관리합니다.

### 섹터(Sector)와 4KB 정렬(Alignment)
<img width="3999" height="2283" alt="image" src="https://github.com/user-attachments/assets/532199be-5b50-49a3-bde0-f26ecc8d5b55" />

과거 HDD는 자기 디스크의 물리적 최소 단위로 **512 Bytes** 섹터를 사용하였습니다. 하지만 데이터 밀도가 높아지면서 오류 교정 효율을 위해 최신 장치(Advanced Format)는 **4KB** 물리 섹터를 채택하고 있습니다.

- **미정렬 문제(Misalignment):** 논리적 파티션의 시작점이 물리 섹터(4KB)의 경계와 일치하지 않으면, 하나의 논리적 쓰기 작업이 두 개의 물리 섹터에 걸쳐 발생하게 됩니다. 이는 입출력 성능을 50% 이상 저하시킬 수 있으므로, SRE는 파티션 생성 시 시작 섹터 번호가 8의 배수(512B 기준) 혹은 4096의 배수인지 반드시 확인해야 합니다.

### SSD와 FTL (Flash Translation Layer)

SSD는 HDD와 달리 덮어쓰기가 불가능하며, 데이터를 지우는 단위(Block)와 쓰는 단위(Page)가 다릅니다.
<img width="720" height="1009" alt="image" src="https://github.com/user-attachments/assets/d78b1475-ba3c-4307-91b2-9d640d36fc23" />

- **FTL:** 커널이 요청하는 LBA(Logical Block Address)를 SSD 내부의 실제 물리 주소(PBA)로 매핑하는 하드웨어 컨트롤러 내의 소프트웨어 계층입니다.
- **Wear Leveling:** 특정 셀의 수명이 다하는 것을 막기 위해 데이터를 골고루 분산하여 저장하는 기술로, 커널의 I/O 스케줄러와 밀접하게 연동됩니다.

---

## 2. 펌웨어 및 파티션 테이블 계층: 구조화된 지도

디스크의 전원을 켰을 때 메인보드의 펌웨어(BIOS/UEFI)가 가장 먼저 읽어 들이는 영역입니다.

### MBR vs GPT 상세 구조

| **특징** | **MBR (Master Boot Record)** | **GPT (GUID Partition Table)** |
| --- | --- | --- |
| **위치** | LBA 0 (첫 번째 섹터) | 디스크 앞부분 및 뒷부분(백업) |
| **주소 지정 방식** | 32-bit (최대 2.2 TB 제한) | 64-bit (최대 9.4 ZB 지원) |
| **중복성** | 없음 (첫 섹터 깨지면 전멸) | 헤더 및 파티션 테이블 복사본 유지 |
| **식별자** | 없음 | 모든 파티션에 고유 UUID(GUID) 부여 |

### UEFI와 Protective MBR

GPT 방식은 과거 BIOS 방식의 도구가 디스크를 빈 것으로 착각하고 파티션을 덮어쓰는 것을 방지하기 위해, LBA 0에 가짜 파티션 정보인 **Protective MBR**을 기록합니다. 이는 하위 호환성과 데이터 안전을 위한 핵심적인 CS 설계입니다.

---

## 3. LVM 계층 (Logical Volume Management): 동적 추상화
<img width="500" height="330" alt="image" src="https://github.com/user-attachments/assets/ac6f444f-17fc-4fcd-bff1-04a2b2bd47b3" />

물리적 파티션의 정적 한계를 극복하기 위해 도입된 커널의 **Device Mapper** 기술입니다.

### Extent: LVM의 원자(Atom)

LVM은 모든 물리 공간을 **PE(Physical Extent)**라는 고정 크기 블록으로 나눕니다.
<img width="1071" height="593" alt="image" src="https://github.com/user-attachments/assets/6badf40a-3aa1-4e92-a2fd-8d0c99ea9931" />

- **매핑 메커니즘:** 사용자가 LV를 사용할 때, 커널은 **LE(Logical Extent)** 번호를 이에 대응하는 **PE** 번호로 실시간 변환합니다.
- **유연성의 근거:** 데이터 이동(`pvmove`)이나 용량 확장(`lvextend`)이 가능한 이유는 커널 레벨에서 이 매핑 테이블의 포인터만 수정하면 되기 때문입니다. 물리적인 데이터 이동 없이도 논리적인 구조 변경이 가능합니다.

### Snapshot과 CoW (Copy-on-Write)

LVM 스냅샷은 생성 시점에 데이터를 복사하지 않습니다. 대신 원본 볼륨의 데이터가 변경되려 할 때, 변경 직전의 데이터를 스냅샷용 별도 공간으로 복사합니다. 이를 통해 최소한의 용량으로 특정 시점의 무결성을 보장합니다.

---

## 4. 파일 시스템 계층 (File System): 메타데이터의 질서

블록 장치 위에 "파일"이라는 추상화된 단위를 구현하는 논리 구조입니다.

### Inode와 데이터 블록의 포인터 구조

리눅스 파일 시스템(ext4, XFS)에서 파일 이름은 디렉터리 파일 내의 매핑 정보일 뿐이며, 실제 모든 정보는 **Inode**에 저장됩니다.
<img width="876" height="588" alt="image" src="https://github.com/user-attachments/assets/c03171e4-9beb-4db7-950c-0cbd1d25d97b" />

- **Direct/Indirect Block:** 작은 파일은 Inode 내의 포인터가 데이터 블록을 직접 가리키지만, 큰 파일은 포인터가 다른 포인터 블록을 가리키는 다단계 구조(Indirect)를 가집니다. 최신 XFS는 이를 효율화하기 위해 **B-tree** 구조를 사용합니다.
- **Journaling의 원리:** 파일 시스템 수정 사항을 실제 데이터 영역에 쓰기 전, **Journal** 영역에 먼저 기록합니다. 이는 시스템 충돌 후 재부팅 시 수천만 개의 파일을 전수 조사(`fsck`)하지 않고 로그만 확인하여 수 초 내에 무결성을 회복하게 해줍니다.

---

## 5. OS 통합 계층: VFS와 시스템 콜

운영체제는 다양한 파일 시스템을 하나의 일관된 인터페이스로 사용자에게 제공합니다.

### VFS (Virtual File System)
<img width="1052" height="1488" alt="image" src="https://github.com/user-attachments/assets/dde257e5-a5ec-4622-91cf-ac1030b0c0c4" />

사용자 프로그램이 `open()`, `read()` 시스템 콜을 호출하면, 커널의 **VFS** 계층이 해당 파일이 위치한 실제 파일 시스템(XFS, ext4, 혹은 네트워크 상의 NFS)의 고유 함수로 이를 연결합니다.

- **Dentry Cache:** 디렉터리 경로를 매번 디스크에서 해석하는 비용을 줄이기 위해, 커널은 경로와 Inode 번호의 매핑 정보를 메모리에 캐싱합니다.

### /etc/fstab의 정교한 매커니즘
<img width="350" height="207" alt="image" src="https://github.com/user-attachments/assets/a051285c-b47d-424a-95bf-1d9b6f9c96af" />

부팅 시 `systemd-fstab-generator`는 이 파일을 읽어 각 마운트 지점을 **systemd unit**으로 변환합니다.

- **UUID의 중요성:** 하드웨어 구성 변경 시 `/dev/sda`가 `/dev/sdb`로 바뀔 수 있으므로, 파일 시스템 생성 시 부여되는 고유 번호인 **UUID**를 사용하는 것이 시스템 안정성(Reliability)의 핵심입니다.
