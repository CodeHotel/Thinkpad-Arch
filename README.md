# Thinkpad-Arch
노트북 아치 리눅스(Arch Linux) 설치/유지보수 일기

시스템소프트웨어와실습, 웹프로그래밍 과목을 수강하면서 리눅스의 편리함, 가벼움, 폭넓은 커스터마이징 기능, 그리고 무엇보다 GUI의 부재와 VIM을 위시로 한 빠른 작업 환경에 매력을 느끼게 되었고, 그 결과 Lenovo Thinkpad X1 Extreme Gen5를 구입해 아치리눅스 설치를 하게 되었다.
이미 설치한지 시간이 2주정도 흘렀지만, 재설치를 하게 되어 나중에 또 재설치를 하게 될 것을 대비해 기록한다.
설치가이드를 의도하고 쓴 것이 아니기 때문에 다른 사람이 읽게 된다면 다소 헷갈릴 수 있으니, 공식 한국어 Arch Wiki 를 참고하는 것을 추천한다.

환경은 다음과 같다:

Lenovo Thinkpad X1 Extreme Gen5
튜닝: Samsung nvme 990 2TB SSD와 32GB*2램으로 업그레이드
->펌웨어가 새로운 메모리에 대한 정보가 없어서 그런지, 처음에 부팅이 되지 않고 Function 키들의 LED가 순서대로 깜빡이기만 했다. 그러나 이는 10분정도 기다리면 인식 완료해 정상적으로 부팅이 되는 문제였음.

과정은 다음과 같다:
1. 이미 설치된 파티션을 싸그리 날리고, 처음부터 파티셔닝을 진행한다
2. 인터넷에 연결하고, 리눅스 커널부터 드라이버까지 아치리눅스를 설치한다.
3. 필요한 각종 프로그램을 설치하며, 데스크탑 환경을 구성한다.
4. 편의성을 위해 기존에 사용하던 컨픽 파일을 불러온다
5. 꾸미기, 편의성 스크립트를 작성한다.


먼저 기존의 파티션은 4개로 나뉘어 있었다. ( fdisk -l 로 조회 가능 )
1. EFI 부팅 시스템(500Mb)-GRUB(부팅매니저)에 의해 사용됨
2. LVM 파일시스템(1TB)-리눅스가 사용
3. 포맷되지 않은 파일시스템(500Gb)-윈도우가 사용할 예정이었음
4. 포맷되지 않은 파일시스템(338.9Gb)-윈도우와 공유할 예정이었음

그 외에 logical volume에는 다음이 있었다:
Disk /dev/mapper/volgroup0-lv_root: 30Gib - 리눅스 시스템(/home 빼고 나머지)
Disk /dev/mapper/volgroup0-lv_home:994GiB - /home 디렉터리

*추후에 재설치하는 경우: logical volume만 삭제*
비활성화:
lvchange -an /dev/volgroup0/lv_root
lvchange -an /dev/volgroup0/lv_home
제거:
lvremove /dev/volgroup0/lv_root
lvremove /dev/volgroup0/lv_home
그룹 제거:
vgremove volgroup0
GRUB파티션 포맷:
mkfs.fat -F32 /dev/nvme0n1p1

*완전한 재설치: 모든 파티션 제거*
먼저 fdisk /dev/nvme0n1(fdisk -l 로 저장장치 이름 볼수있음)를 실행하고, d명령어로 1~4파티션을 날렸다.
![WIN_20231117_20_08_21_Pro](https://github.com/CodeHotel/Thinkpad-Arch/assets/89632518/b8c655e7-d3eb-40d2-8f3e-0545481f9f4b)
이제 nvme0n1(SSD)안에 아무것도 없다. 파티셔닝을 시작한다.

![WIN_20231117_20_19_57_Pro](https://github.com/CodeHotel/Thinkpad-Arch/assets/89632518/a38f80c5-5d83-4dd0-a0e1-c52b12704a25)

fdisk /dev/nvme0n1

EFI 레이블(GRUB 부트로더용) 만들기:
p (이후 출력되는 파티션이 없는지, 즉 다 지워졌는지 확인)
g (레이블 추가)
n (파티션 추가-엔터 두번으로 기본값 사용, 마지막에 +500M으로 크기 지정)
->여기서 vfat signature이 이미 존재한다고 하는데 Y를 입력해 삭제
t 후 1 (파티션 타입 EFI로 세팅)

![image](https://github.com/CodeHotel/Thinkpad-Arch/assets/89632518/ae889c36-7f46-4635-908c-ab7df3d9f90c)
메인 리눅스 파티션 만들기:
> 파티션 추가
  n(추가)
  엔터엔터(번호,시작주소 기본값 사용)
  +1T(1테라 용량 지정)
  Y(기존에 존재하던 LVM2 시그니쳐 삭제)
> LVM타입지정
  t
  2
  44(리눅스 LVM)

공유 파티션, 윈도우용 파티션 만들기:
위와 동일하게 진행하나, 공유 파티션은 +500G, 타입은 11(Microsoft basic data)로 하였고
윈도우 파티션은 마지막 파티션인 만큼 끝주소를 지정하지 않았다.(자동으로 남은 모든 공간을 할당함)
윈도우 파티션도 마찬가지로 타입번호 11을 부여하였다.
![image](https://github.com/CodeHotel/Thinkpad-Arch/assets/89632518/c0a0d9bd-309d-4f14-8ad9-c00165cd1aeb)
p를 눌러 확인하면 다음과 같다.
w를 눌러 저장, 적용한다.

다음으로 다시 터미널로 나와 포맷을 진행한다.

![image](https://github.com/CodeHotel/Thinkpad-Arch/assets/89632518/ad6ba12b-bc3e-4ad6-9182-7309a8965441)

EFI 부팅 파티션을 fat32로 포맷한다
> mkfs.fat -F32 /dev/nvme0n1p1

공유 파티션을 exfat으로 포맷한다
> mkfs.exfat /dev/nvme0n1p3

![image](https://github.com/CodeHotel/Thinkpad-Arch/assets/89632518/5774ffb4-9edd-40fb-a937-85cab00715f9)
LVM 파티션을 세팅한다
Physical Volume 생성
> pvcreate --dataalignment 1m /dev/nvme0n1p2
볼륨그룹 생성
> vgcreate volgroup0 /dev/nvme0n1p2
lv_root(리눅스 시스템 파티션) 생성
> lvcreate -L 50GB volgroup0 -n lv_root
lv_home(리눅스 홈 파티션) 생성
> lvcreate -l 100%FREE volgroup0 -n lv_home

생성 확인
> modprobe dm_mod
> vgscan
logical volume 활성화
> vgchange -ay

포맷
> mkfs.ext4 /dev/volgroup0/lv_root
> mkfs.ext4 /dev/volgroup0/lv_home

마운팅
> mount /dev/volgroup0/lv_root /mnt
> mkdir
