# Arch Linux 開機加速

## 說明

雖然是在Arch Linux上實驗的，但想法都能套用在其他發行版上



## 要點

減少kernel和initramfs的大小、刪減不必要的操作，減肥減肥再減肥



## 測量

```systemd-analyze```列出各個階段所花的時間

在我的環境下有```firmware```, ```loader```, ```kernel```, ```userspace```四項，其中：

* ```firmware```: UEFI的部分

* ```loader```: GRUB的部分

* ```kernel```: 從kernel開始initialize包含initramfs到systemd開始前的部分

* ```userspace```: systemd所花費的時間

```systemd-analyze blame``` 可以列出systemd各項service花費的時間

```systemd-analyze critical-chain``` 依相依性列出各項service啟動的時間點和花費的時間

```sudo journalctl | grep Startup``` 這樣比較方便比較各次修改的結果



## UEFI

這大概是最簡單的部分吧

把不必要的功能關掉，然後開機順序保持目標在第一順位

開機順序大多是影響最大的，尤其是網卡PXE的部分不需要就該移到後面



## GRUB

```loader```有包含從硬碟載入kernel和initramfs的時間，這部分影響最大，見下方*Linux kernel*和*initramfs*的部分

如果有多個entry，把目標設為預設，開機時壓著方向右或是```<enter>```就能消除人為的測量誤差

如果只有一個系統就直接把```GRUB_TIMEOUT```設為0吧

如果不在意外觀也可以不用圖形化的介面

再來影響小很多(但通常還是測的出來)的就是減少載入的mod

在UEFI下預設會載入GPT和MBR兩種table的mod，如果不需要用MBR就在```/etc/default/grub```把他拿掉

顯示部分的mod在UEFI底下預設會載入```video.lst```列出的四種：

* ```efi_uga```: 舊EFI時代的顯示方式

* ```efi_gop```: UEFI大多是這個

* ```video_bochs```: bochs模擬器

* ```video_cirrus```: qemu常用的video

用```/etc/default/grub```的```GRUB_VIDEO_BACKEND```可以指定，如果有就只會載入指定的

修改```/etc/default/grub```後都需要手動```sudo grub-mkconfig -o /boot/grub/grub.cfg```才會套用，詳見```/etc/grub.d/00_header```

font觀察```/boot/grub/grub.cfg```，預設是載入Arch Linux的/usr裡面的，但可以發現另一個是直接```loadfont unicode```，直接搜尋```/boot/grub/font```裡面的，修改成這種方式可以跳過載入額外的分區



## Linux kernel

自行針對自己的硬體編譯Linux kernel，可以減少kernel image的大小，進而減少GRUB載入kernel的時間，少一些部件需要initialize也會讓kernel啟動更快，甚至可以增進一點效能

在config時除了需不需要某項功能，還要考慮要將他built-in或者編譯成module，會需要的功能可以用官方kernel看自動載入了哪些modules再做刪減

built-in的話在開機時就會initialize，會增加開機時間，module時則是在module載入時才會運行

我的建議是將掛載root分區和基本的顯示及鍵盤會需要的功能都built-in，比如說sata, ext4, efifb, atkbd等，其餘有需要的再編成module，比如說顯卡、網卡、藍芽、網路功能、iptables，甚至usb等，交由systemd自動載入，以減少kernel image大小

一些安全性的保護措施可以根據實際應用需求做衡量，這部分大多數都是無法編譯成modules的

除此之外還要衡量kernel和initramfs的壓縮方式，有xz, bzip2, gzip, lzo, lz4, lzma，其中xz壓縮率最高但解壓慢，lzo壓縮率最低但解壓極快，壓縮時的速度不需要考慮，各項可能需要分別試驗衡量一下

根據我的觀察kernel的解壓縮時間應該沒有被測量到，要注意

編譯器的optimization level如果是Os或O2可以在config內調整，其餘需要編輯Makefile，搜尋O2去修改，常見有Os, O2, O3, Ofast，其中用Ofast跑分會小幅高一點，O2是預設值，Os能在和O2效能差不多的情況下把大小壓小，O3和Ofast會顯著增加大小

Makefile的CFLAGS可以針對cpu加上```-mtune=<cpu-type>```來指定可用的指令集範圍，如果是generic在大多數機子上都能跑，native則是偵測當前機子上的，詳見gcc說明書

config需要特別注意的是hz的設定也可能會影響開機速度，建議保持預設的1000hz

config的方法常見的有```make menuconfig```和```make nconfig```，後者比較新

編譯時別忘了加```-j<n>```，其中n為最大同時jobs數，個人習慣實體核心數\*5，如果有-j後面不加數量就是不限制，gui或terminal可能會暫時當掉

如果在別的地方編譯```make modules_install```可以用```INSTALL_MOD_PATH```變數指定複製到的路徑，方便把modules給分出來

如果編譯後還想再刪減，但不知道該從哪裡下刀可以用```ls -l */built-in.o```查看各個類別的大小

建議在桌機上編譯



## initramfs

針對自己編譯的kernel製作initramfs，減少initramfs的大小，進而降低bootloader載入initramfs和kernel解壓縮initramfs的時間

initramfs是透過cpio及xz, bzip2, gzip, lzo, lz4, lzma其中一個壓縮過後的userspace，可以自己打包，但求方便和功能完整還是用Arch特有的```mkinitcpio```和```lsinitcpio```處理

### mkinitcpio

```mkinitcpio -p <preset name>```是打包時用的指令

```lsinitcpio <initramfs>```會列出initramfs裡的所有檔案，加個```-x```會解壓縮到當前路徑，可以用來觀察內容

可以由原本的/etc/mkinitcpio.conf和/etc/mkinitcpio.d/linux.preset複製一份出來修改，保留原本的initramfs-linux.img供出錯時使用

#### FILES

如果kernel有built-in的功能需要firmware，在initramfs階段就需要提供，可以透過這一項加入

#### HOOKS

在conf中可以設定需要的hooks，如果掛載root分區需要的模組都是built-in的，一般只會需要```base```就能開機，但建議還是加個```fsck```在掛載前自動檢查分區

hooks的定義在/lib/initcpio/install/ (/libs/initcpio/hooks/是initramfs裡的模組，透過hooks加入initramfs)

```autodetect```會依據當前系統的需求調整其餘hooks安裝的檔案，例如如果在```fsck```前有```autodetect```，就只會加入機子上root分區檔案系統的對應fsck工具

```strip```會strip目前已經加入的library和binary，刪除不必要的除錯用資訊以減少大小

所以，一般建議hooks為```(base autodetect fsck strip)```

### compiling binaries

用lsinitcpio或是觀察hooks的腳本可以發現，裡面含有的binaries，以上方的hooks設定為例通常會有```busybox```(來自mkinitcpio-busybox的版本), ```mount```, ```switch_root```, ```blkid```, ```fsck```, ```e2fsck```，都是由當前檔案系統複製出來的版本，使用了glibc，對於initramfs通常musl是更好的選擇，大小小很多，效能差異也不大，glibc特有的功能在initramfs通常也都用不到，透過把binaries換成用musl自行編譯的版本會達到很棒的效果，在自行編譯時，也可以順便針對機子加入```-mtune=<cpu-type>```或是利用```-Os```，甚至自行斟酌static link或dynamic link來進一步減少binaries的大小

musl在Arch的package就叫做```musl```，裡面除了musl libc本身，還含有```musl-gcc```指令(gcc的wrapper)，方便使用musl編譯，通常在configure時把變數CC設為musl-gcc，或是編譯時到Makefile將CC設為musl-gcc即可

如果需要musl的ldd，例如在修改mkinitcpio找尋需要的libraries的部份，或是觀察linking，直接執行/lib/ld-musl-\*.so即可，也可以把它link成musl-ldd之類的方便使用

#### [busybox](https://busybox.net/downloads/)

busybox的config方式是採用Kconfig，執行```make menuconfig```就能調整需要的功能

觀察mkinitcpio的init腳本就能找出所有需要的指令，可以減少到只剩需要的功能，也可以留下一些基本的指令用來除錯

busybox的```mount```, ```switch_root```, ```fsck```實做功能已經足夠在initramfs使用，可以改用busybox的實做減少實際需要的binaries數量

其中busybox的mount能判別出ext系列，但沒有分辨是ext2, ext3或ext4，會從ext2開始試著mount，建議cmdline加入rootfstype直接指定

busybox對於```blkid```就比較不全面了，只能查詢UUID，建議還是用一般的blkid

其中比較需要注意的是long options的支援，musl的getopt不會找尋non-option後方的option，但是getopt_long會，所以建議要打開，會用到的例子是init在mount的時候是執行```mount -t <type> <dev> <dist> -o <options>```，如果用getopt會抓不到後面的-o項，造成busybox檢查args的數量時出錯 (getopt在這種情況下的表現其實POSIX沒有定義到)

#### [util-inux](https://git.kernel.org/pub/scm/utils/util-linux/util-linux.git/)

提供```blkid```

#### [e2fsprogs](https://git.kernel.org/pub/scm/fs/ext2/e2fsprogs.git/)

提供```e2fsck```，即```fsck.ext4```,```fsck.ext3```, ```fsck.ext2```



## systemd

### services

首先當然是停用不必要的service，以```sudo systemctl disable```停用，列出已經啟用的service最簡單的方式就是利用bash-completion在disable後方按兩次```<tab>```鍵

再來觀察```systemd-analyze blame```，不必要且沒有明確enable但自動載入的可以用```sudo systemctl mask```擋掉

### journal flush

如果系統用久了就會發現```systemd-journal-flush.service```占很大的時間，但如果停用它journal就會留在/run而不是移到/var，關機了就沒了，解決方法是停用他但在其他時機手動flush，或是限制journal的大小避免花費太多時間，見```/etc/systemd/journald.conf```

### remount

如果把fsck交由initramfs實行，systemd remount一次root就顯得多餘了，確定cmdline有rootflags=rw等想要的mount flags後可以把/etc/fstab的root註解掉避免remount

### mandb

現在Arch的作法是每隔12h開機時更新一次mandb，但這通常很慢，一個作法是改回原本的方法，利用pacman hooks在有更動到manpages時更新，不過這把時間轉嫁給了pacman，在AUR上有mandb-ondemand，同樣是個pacman hook但利用systemd在背景執行不須等待，安裝它後mask man-db.service即可



## quiet boot

解決大部分的東西後，message也顯得不太重要了，vga console或framebuffer console的效能都不佳，甚至可能成為瓶頸，建議停用一些message來加速開機

### kernel console printk

直接在config中把printk的功能拿掉或許會減少很多空間，但這樣就很難debug了，建議留著，讓他只把錯誤訊息輸出在console就好

確定kernel的cmdline有```quiet```這項，可以透過```/proc/cmdline```檢查，從```/etc/default/grub```的```GRUB_CMDLINE_LINUX```修改

### systemd boot message

如果cmdline有了quiet，systemd就只會在出錯時開始輸出，但有時候還是會有不需要又關不掉的功能的報錯，比如說找不到autofs4的module之類的，這時候還是可以在cmdline加入```systemd.show_status=false```強制關掉



關了這些後，理想狀況下linux的開機輸出就差不多只剩initramfs的fsck了，就把他留著吧
