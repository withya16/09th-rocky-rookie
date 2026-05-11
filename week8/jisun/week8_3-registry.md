# 이미지 검색·다운로드·삭제·관리

## 이미지 관리의 생명주기
```
[ 원격 저장소 (Registries) ]          [ 로컬 호스트 (Local Host) ]
 (quay.io, docker.io, etc.)
           |
           | 1. 검색 (search)
           v
+--------------------------+          +---------------------------+
|    이미지 레지스트리       |          |    로컬 이미지 저장소       |
|  (Container Registry)    |          |   (/var/lib/containers)   |
+------------|-------------+          +-------------^-------------+
             |                                      |
             | 2. 다운로드 (pull) -------------------+
             |                                      |
             |                                      | 3. 관리/삭제 (images, rmi)
             |                                      v
             |                        [ Image A ] [ Image B ] [ Image C ]
```

### 이미지 검색
``` bash
$ podman search nginx
```
- 원격 레지시트리에서 저장된 이미지 목록 탐색  
    - Dcoker : Docker Hub만 검색
    - Podman : 여러 저장소 동시 검색 (`/etc/containers/registries.conf` 파일에 등록된 레지스트리 목록 조회)  
      - `docker.io` : Docker Hub 서비스의 기술적 호스트 주소
      - `quay.io` : 레드햇에서 운영하는 컨테이너 레지스트릐 서비스 


### 이미지 다운로드
``` bash
$ podman pull quay.io/nginx:latest
```
- 이미지는 하나의 거대한 파일이 아니라 여러 개의 레이어로 구성  
  -> `pull` 시 중복되는 레이어를 제외하고 필요한 레이어만 받음

### 이미지 목록 및 상세 관리
``` bash
$ podman images
```
- 목록 확인 : 로컬 스토리지(`/var/lib/containers`)에 존재하는 이미지의 ID, 생성일, 용량 확인  

``` bash
$ podman inspect [Image ID]
```
- 상세 분석 : 이미지가 내부적으로 사용하는 포트, 환경변수, 실행명령(CMD), 레이어 정보 등을 Json 형식으로 출력

### 이미지 삭제
``` bash
$ podman rmi [Image ID]
```
- 실행 중이거나 정지된 컨테이너가 이미지 참조 중일 시 삭제 거부

## 이미지 구조
```
[ 물리적 저장소 (Host Disk) ]              [ 논리적 결합 (Kernel OverlayFS) ]          [ 컨테이너 내부 뷰 ]
/var/lib/containers/storage/
                                         (사용자 작업 공간)
├── overlay/ (UpperDir)  ----------------> [ Writable Layer ]  --------------+
│   (컨테이너 생성 시 생성)                    (실시간 변경분 저장)                |
│                                                                            |
│                                        (읽기 전용 이미지)                   |    / (Root FS)
├── overlay-layers/ (LowerDir) ----------> [ Layer 3 (Config) ]  ------------|--> [ Merged View ]
│   (이미지별 고유 해시 폴더)                   [ Layer 2 (App)    ]  ------------|    (하나의 파일시스템
│                                          [ Layer 1 (Base OS) ]  ------------+     처럼 통합되어 보임)
│
├── overlay-images/ (Metadata)
│   (이미지-레이어 매핑 정보 DB) ───────────┐
│                                         │ (연결 고리 관리)
└── storage.conf (Configuration) ─────────┘
```
### 기반 데이터 : `overlay-layers`
- 원격 저장소에서 내려받은 원본 데이터 
- 읽기 전용

### 변경 데이터 : `overlay/`
- 컨테이너가 실행된 후 사용자가 파일을 새로 만들거나 수정할 때 발생하는 차이점만 저장
- 컨테이너 삭제 시 이 디렉토리도 삭제됨

### 통합 엔진 : `overlay-images` + `storage.conf`
- `overlay-images` : 특정 이미지를 만들려면 어떤 레이어 조각들을 순서대로 쌓아야 하는지의 방법 포함
- `storage.conf` : overlay-images를 기반으로 만들 환경 정의 