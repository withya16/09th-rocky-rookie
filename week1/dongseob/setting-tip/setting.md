https://wallbuster.tistory.com/10
기준으로 설정했으며 아래는 위 게시글과 다른 부분에 대해 추가 작성한 것

[VM Setting]
1. RHEL 9.6 버전 사용 + DVD.iso 설치
2. Disk : 80 GB
3. Processors : 4
4. Memory : 4GB

[RHEL Setting]
1. Software Selection에서 "Development Tools"만 체크
2. 파티션 수동 설정(순서 중요)
/boot/efi 600 MiB
/boot 1 GiB
swap 4GiB
/home 15GiB
/ (나머지-걍 공백으로 두고 추가하시면 됩니다)
3. Network & Hostname - 저는 일단 그냥 기본세팅 그대로 "자동"으로 했습니다.
호스트명 : rhel9
4. root 비밀번호 -> 옵션 체크하는 거 2개 다 미체
5. 사용자  생성 -> dongseob 유저 1개 생

[Subscription]
1. 설정 후 재부팅하신 다음 root 계정으로,  바로 "Register Subscription"이라고 상단에 뜹니다.
2. 그거 누르거나 설정 들어가셔서 "정보" 들어가시고 제일 하단에 보시면 "Subscription"에 "System Not Registered"라고 표시되어 있을 건데 그거 클릭
3. 제일 아래에 "Registeration Type"에서 "Red Hat Account" 체크 -> "Login"과 "암호"에 본인 계정 거 입력하시고 우측 상단에 "Register" 클릭
4. 터미널 키셔서 제 캡쳐본에 나와있는대로 subscription-manager list, subscription-manager register, subscription-manager identity 로 정보 확인하셔서 제 캡쳐본이랑 비슷하게 결과 출력되면 괜춘
5. subscription-manager register 하시면 저처럼 "비활성화됨"이라고 뜰텐데 출력된 결과처럼 SCA 방식을 써서 그렇다고 하네요. 문제 없다고 합니당
6. dnf repolist 하시면 결과처럼 2개 : AppStream, BaseOS 가 떠야 됩니다.
7. 만약 안 뜨면 Red Hat Developer 들어가셔서 서브스크랩션 활성화에서 "Active" 버튼 눌러주세용
8. 그 후 sudo dnf update -y 하신 후에 스냅샷 하나 찍어두시면 됩니다 -끗-
