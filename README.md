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
`lvchange -an /dev/volgroup0/lv_root`
`lvchange -an /dev/volgroup0/lv_home`
제거:
`lvremove /dev/volgroup0/lv_root`
`lvremove /dev/volgroup0/lv_home`
그룹 제거:
`vgremove volgroup0`
GRUB파티션 포맷:
`mkfs.fat -F32 /dev/nvme0n1p1`

*완전한 재설치: 모든 파티션 제거*
먼저 `fdisk /dev/nvme0n1`(fdisk -l 로 저장장치 이름 볼수있음)를 실행하고, d명령어로 1~4파티션을 날렸다.
![WIN_20231117_20_08_21_Pro](https://github.com/CodeHotel/Thinkpad-Arch/assets/89632518/b8c655e7-d3eb-40d2-8f3e-0545481f9f4b)
이제 nvme0n1(SSD)안에 아무것도 없다. 파티셔닝을 시작한다.

![WIN_20231117_20_19_57_Pro](https://github.com/CodeHotel/Thinkpad-Arch/assets/89632518/a38f80c5-5d83-4dd0-a0e1-c52b12704a25)

`fdisk /dev/nvme0n1`

EFI 레이블(GRUB 부트로더용) 만들기:
p (이후 출력되는 파티션이 없는지, 즉 다 지워졌는지 확인)
g (레이블 추가)
n (파티션 추가-엔터 두번으로 기본값 사용, 마지막에 +500M으로 크기 지정)
->여기서 vfat signature이 이미 존재한다고 하는데 Y를 입력해 삭제
t 후 1 (파티션 타입 EFI로 세팅)

![image](https://github.com/CodeHotel/Thinkpad-Arch/assets/89632518/ae889c36-7f46-4635-908c-ab7df3d9f90c)
메인 리눅스 파티션 만들기:
- 파티션 추가
  n(추가)
  엔터엔터(번호,시작주소 기본값 사용)
  +1T(1테라 용량 지정)
  Y(기존에 존재하던 LVM2 시그니쳐 삭제)
- LVM타입지정
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
`mkfs.fat -F32 /dev/nvme0n1p1`

공유 파티션을 exfat으로 포맷한다
`mkfs.exfat /dev/nvme0n1p3`

![image](https://github.com/CodeHotel/Thinkpad-Arch/assets/89632518/5774ffb4-9edd-40fb-a937-85cab00715f9)
LVM 파티션을 세팅한다
Physical Volume 생성
`pvcreate --dataalignment 1m /dev/nvme0n1p2`
볼륨그룹 생성
`vgcreate volgroup0 /dev/nvme0n1p2`
lv_root(리눅스 시스템 파티션) 생성
`lvcreate -L 50GB volgroup0 -n lv_root`
lv_home(리눅스 홈 파티션) 생성
`lvcreate -l 100%FREE volgroup0 -n lv_home`

생성 확인
`modprobe dm_mod`
`vgscan`
logical volume 활성화
`vgchange -ay`

포맷
`mkfs.ext4 /dev/volgroup0/lv_root`
`mkfs.ext4 /dev/volgroup0/lv_home`

마운팅
`mount /dev/volgroup0/lv_root /mnt`
`mkdir /mnt/home`
`mount /dev/volgroup0/lv_home /mnt/home`
`mkdir /mnt/etc`
`genfstab -U -p /mnt `/mnt/etc/fstab`
`cat /mnt/etc/fstab` 로 결과 확인


다음으로 리눅스 설치를 진행한다.


WIFI연결
wlan 매니저 실행:
`iwctl`
어댑터 찾기:
`device list`
와이파이 검색:
`station wlan0 scan`
검색된 와이파이 보기:
`station wlan0 get-networks`
특정 와이파이에 연결:
`station wlan0 connect 와이파이이름`
`quit`

Ping으로 연결 테스트
`ping -c 5 8.8.8.8`



아치리눅스 기초패키지 설치
`pacstrap -i /mnt base`

root디렉터리를 실제 하드디스크로 변경
`arch-chroot /mnt`

리눅스 설치:
`pacman -S linux linux-headers linux-firmware`
(linux-lts를 설치해도 되지만 rtx 3060드라이버와 충돌현상이 있는 듯함)
->만일 이 과정에서 PGP관련 에러가 뜬다면
`pacman-key --init`
`pacman-key --populate archlinux`
를 실행한후 다시 하면 된다.

WIFI관련 패키지 설치
`pacman -S networkmanager wpa_supplicant wireless_tools netctl`
`systemctl enable NetworkManager`

LVM 파티션 지원 추가
`pacman -S lvm2`
![WIN_20231117_23_17_44_Pro](https://github.com/CodeHotel/Thinkpad-Arch/assets/89632518/030611ce-bba0-4bde-9842-c788bba681aa)
/etc/mkinitcpio.conf 파일의 HOOKS에 block과 filesystems사이 lvm2 추가
설정 적용
`mkinitcpio -p linux`


지역 설정
/etc/locale.gen 파일
#ko_KR.UTF-8 UTF-8 에서 # 지우기
적용:
`locale-gen`


root유저 패스워드 설정
`passwd`
사용자 추가
`useradd -m -g users -G wheel 유저네임`
`passwd 유저네임`

sudo설치
`pacman -S sudo`

사용자 권한 부여
`EDITOR=vim visudo`
![image](https://github.com/CodeHotel/Thinkpad-Arch/assets/89632518/1db2ed4d-4fd4-464a-8f78-a6cb7ce3bc96)
wheel라인 # 삭제 후 저장


*GRUB설치*
`pacman -S grub efibootmgr dosfstools os-prober mtools`
`mkdir /boot/EFI`
`mount /dev/nvme0n1p1 /boot/EFI`
`grub-install --target=x86_64-efi --bootloader-id=grub_uefi --recheck`

`ls -l /boot/grub/locale`
존재하지 않는다면
`mkdir /boot/grub/locale`

`cp /usr/share/locale/en\@quot/LC_MESSAGES/grub.mo /boot/grub/locale/en.mo`
`grub-mkconfig -o /boot/grub/grub.cfg`


재부팅
`exit`
`reboot`(혹은 `shutdown`)

이 때 shutdown후 usb를 제거하고 다시 전원을 켜도 정상적으로 부팅이 되어야 함.


시간설정
`sudo timedatectl list-timezones`
한국은 Asia-Seoul임
`sudo timedatectl set-timezone Asia/Seoul`
`sudo systemctl enable systemd-timesyncd`

기기 이름 설정
`hostnamectl set-hostname 기기이름`
`/etc/hosts` 파일 설정
`sudo vim /etc/hosts`

127.0.0.1 localhost
127.0.1.1 기기이름
추가 후 저장

다시 wifi에 연결
`nmcli device wifi list`
`nmcli device wifi connect iptime(실제 이름으로 대체) password 비밀번호`

ucode설치
`sudo pacman -S intel-ucode`

pacman.conf에서 multilib 허용
/etc/pacman.conf
[multilib]
Include = /etc/pacman.d/mirrorlist
코멘트 해제

내장그래픽 드라이버 설치
`sudo pacman -S mesa mesa-demos mesa-utils lib32-mesa` (인텔 내장그래픽)
사운드 드라이버 설치
`sudo pacman -S alsa-utils alsa-plugins alsa-lib alsa-firmware sof-firmware`
`echo "options snd-intel-dspcfg dsp_driver=1" | sudo tee -a /etc/modprobe.d/alsa-base.conf` (펌웨어 버그 때문에 다운그레이드시켜야함)
`alsamixer` 에서 사운드카드 인식되는지 확인(f6 후 선택, 이후 볼륨설정)
`speaker-test -c 2 -t wav` 로 테스트

**설치 중 오류가 뜬다면 `rm /var/lib/pacman/db.lck` 실행

nvidia 드라이버 설치
`sudo pacman -S nvidia nvidia-prime nvidia-settings nvidia-utils`

배터리 절약 기능
`sudo pacman -S tlp`
`sudo systemctl enable --now tlp`

GUI 설치
xorg 설치
`sudo pacman -S xorg-server xorg-xinit xorg-xrandr xorg-xev`

i3 설치
`sudo pacman -S i3`
전부 설치
`noto-fonts 선택`

alacritty 설치
`sudo pacman -S alacritty`
`export TERMINAL=alacritty`

GUI정상작동확인
`echo "exec i3" `~/.xinitrc`
`startx` 실행하여 UI환경이 실행되는지 확인
`nvidia-smi` 실행하여 오직 Xorg만 딱 4MiB만큼만 실행중인지 확인

해상도 설정
`xrandr` 로 가능한 해상도와 현재 디스플레이 장치 확인
`xrandr --output eDP-1 --mode 1920x1080` 로 해상도 바르게 설정(eDP-1이 아닌 다른 디스플레이 장치일 경우 선택)

밝기 설정
`sudo pacman -S brightnessctl`
`brightnessctl s 20%+` 및
`brightnessctl s 20%-` 로 확인

fn 밝기/사운드키 테스트
~/.config/i3/config 파일 복사 덮어씌우기

블루투스 설치
`sudo pacman -S pulseaudio-bluetooth bluez blueman`

Arch 오우너의 필수템 neofetch설치
`sudo pacman -S neofetch`

yay설치
`pacman -S --needed git base-devel`
`cd /opt/`
`sudo git clone https://aur.archlinux.org/yay.git`
`sudo chown -R 유저네임 yay`
`cd yay`
`makepkg -si`

브라우저와 Jetbrains 설치
`yay -S google-chrome`
`yay -S jetbrains-toolbox`
`jetbrains-toolbox & disown`
툴박스에서 intelliJ 설치

한글 입출력 설정
*Arch 공식 위키에서도 UIM Byeoru를 권장하고 있으나, IntelliJ와 호환이 잘 되지 않는 것 같아 fcitx를 사용하겠음*
`sudo pacman -S noto-fonts-cjk fcitx5 fcitx5-hangul fcitx5-gtk fcitx5-qt fcitx5-config-qt`
`fcitx5 &`
상태바 우클릭 -> 왼쪽에 Hangul English 키보드가 모두 있도록 설정 -> Global Options에서 한영키로 세팅(Ralt)
구글크롬에서 제대로 되는지 확인

한영키를 Switch_mode로 매핑
`echo "keycode 108 = Mode_switch" > ~/.Xmodmap`

~/.xinitrc 파일을 다음과 같이 설정
`export GTK_IM_MODULE=fcitx5`
`export QT_IM_MODULE=fcitx5`
`export XMODIFIERS=@im=fcitx5`
`exec i3`

*이 세줄은 exec i3전에 와야 하며, fcitx가 아닌 fcitx5로 쓰는것이 중요함.*

