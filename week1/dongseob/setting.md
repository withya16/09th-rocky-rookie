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
