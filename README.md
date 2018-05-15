# Arch Linux 開機加速

## 說明

雖然是在 Arch Linux 上實驗的，但想法都能套用在其他發行版上



## 要點

減少 kernel 和 initramfs 的大小、刪減不必要的操作，減肥減肥再減肥



## 測量

```systemd-analyze``` 列出各個階段所花的時間

在我的環境下有 ```firmware```, ```loader```, ```kernel```, ```userspace``` 四項，其中：

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

開機順序大多是影響最大的，尤其是網卡 PXE 的部分不需要就該移到後面



## GRUB

```loader``` 有包含從硬碟載入 kernel 和 initramfs 的時間，這部分影響最大，見下方 *Linux kernel* 和 *initramfs* 的部分

如果有多個 entry ，把目標設為預設，開機時壓著方向右或是 ```<enter>``` 就能消除人為的測量誤差

如果只有一個系統就直接把 ```GRUB_TIMEOUT``` 設為 0 吧

如果不在意外觀也可以不用圖形化的介面

再來影響小很多(但通常還是測的出來)的就是減少載入的 mod

在 UEFI 下預設會載入 GPT 和 MBR 兩種 table 的 mod ，如果不需要用 MBR 就在 ```/etc/default/grub``` 把他拿掉

顯示部分的 mod 在 UEFI 底下預設會載入 ```video.lst``` 列出的四種：

* ```efi_uga```: 舊 EFI 時代的顯示方式

* ```efi_gop```: UEFI 大多是這個

* ```video_bochs```: bochs 模擬器

* ```video_cirrus```: qemu 常用的 video

用 ```/etc/default/grub``` 的 ```GRUB_VIDEO_BACKEND``` 可以指定，如果有就只會載入指定的

修改 ```/etc/default/grub``` 後都需要手動 ```sudo grub-mkconfig -o /boot/grub/grub.cfg``` 才會套用，詳見 ```/etc/grub.d/00_header```

font觀察 ```/boot/grub/grub.cfg``` ，預設是載入 Arch Linux 的 /usr 裡面的，但可以發現另一個是直接 ```loadfont unicode``` ，直接搜尋 ```/boot/grub/font``` 裡面的，修改成這種方式可以跳過載入額外的分區



## Linux kernel

自行針對自己的硬體編譯 Linux kernel ，可以減少 kernel image 的大小，進而減少 GRUB 載入 kernel 的時間，少一些部件需要 initialize 也會讓 kernel 啟動更快，甚至可以增進一點效能

在 config 時除了需不需要某項功能，還要考慮要將他 built-in 或者編譯成 module ，會需要的功能可以用官方 kernel 看自動載入了哪些 modules 再做刪減

built-in 的話在開機時就會 initialize ，會增加開機時間， module 時則是在 module 載入時才會運行

我的建議是將掛載 root 分區和基本的顯示及鍵盤會需要的功能都 built-in ，比如說 sata, ext4, efifb, atkbd 等，其餘有需要的再編成 module ，比如說顯卡、網卡、藍芽、網路功能、 iptables ，甚至 usb 等，交由 systemd 自動載入，以減少 kernel image 大小

一些安全性的保護措施可以根據實際應用需求做衡量，這部分大多數都是無法編譯成 modules 的

除此之外還要衡量 kernel 和 initramfs 的壓縮方式，有 xz, bzip2, gzip, lzo, lz4, lzma ，其中 xz 壓縮率最高但解壓慢， lzo 壓縮率最低但解壓極快，壓縮時的速度不需要考慮，各項可能需要分別試驗衡量一下

根據我的觀察 kernel 的解壓縮時間應該沒有被測量到，要注意

編譯器的 optimization level 如果是 -Os 或 -O2 可以在 config 內調整，其餘需要編輯 Makefile ，搜尋 -O2 去修改，常見有 -Os, -O2, -O3, -Ofast ，其中用 -Ofast 跑分會小幅高一點， -O2 是預設值， -Os 能在和 -O2 效能差不多的情況下把大小壓小， -O3 和 -Ofast 會顯著增加大小

Makefile 的 CFLAGS 可以針對 cpu 加上 ```-mtune=<cpu-type>``` 來指定可用的指令集範圍，如果是 generic 在大多數機子上都能跑， native 則是偵測當前機子上的，詳見 gcc 說明書

config 需要特別注意的是 hz 的設定也可能會影響開機速度，建議保持預設的 1000hz

config 的方法常見的有 ```make menuconfig``` 和 ```make nconfig``` ，後者比較新

編譯時別忘了加 ```-j<n>``` ，其中 n 為最大同時 jobs 數，個人習慣實體核心數 \*5 ，如果有 -j 後面不加數量就是不限制， gui 或 terminal 可能會暫時當掉

如果在別的地方編譯 ```make modules_install``` 可以用 ```INSTALL_MOD_PATH``` 變數指定複製到的路徑，方便把 modules 給分出來

如果編譯後還想再刪減，但不知道該從哪裡下刀可以用 ```ls -l */built-in.o``` 查看各個類別的大小

建議在桌機上編譯



## initramfs

針對自己編譯的 kernel 製作 initramfs ，減少 initramfs 的大小，進而降低 bootloader 載入 initramfs 和 kernel 解壓縮 initramfs 的時間

initramfs 是透過 cpio 及 xz, bzip2, gzip, lzo, lz4, lzma 其中一個壓縮過後的 userspace ，可以自己打包，但求方便和功能完整還是用 Arch 特有的 ```mkinitcpio``` 和 ```lsinitcpio``` 處理

### mkinitcpio

```mkinitcpio -p <preset name>``` 是打包時用的指令

```lsinitcpio <initramfs>``` 會列出 initramfs 裡的所有檔案，加個 ```-x``` 會解壓縮到當前路徑，可以用來觀察內容

可以由原本的 /etc/mkinitcpio.conf 和 /etc/mkinitcpio.d/linux.preset 複製一份出來修改，保留原本的 initramfs-linux.img 供出錯時使用

#### FILES

如果 kernel 有 built-in 的功能需要 firmware ，在 initramfs 階段就需要提供，可以透過這一項加入

#### HOOKS

在 conf 中可以設定需要的 hooks ，如果掛載 root 分區需要的模組都是 built-in 的，一般只會需要 ```base``` 就能開機，但建議還是加個 ```fsck``` 在掛載前自動檢查分區

hooks 的定義在 /lib/initcpio/install/ ( /libs/initcpio/hooks/ 是 initramfs 裡的模組，透過 hooks 加入 initramfs )

```autodetect``` 會依據當前系統的需求調整其餘 hooks 安裝的檔案，例如如果在 ```fsck``` 前有 ```autodetect``` ，就只會加入機子上 root 分區檔案系統的對應 fsck 工具

```strip``` 會 strip 目前已經加入的 library 和 binary ，刪除不必要的除錯用資訊以減少大小

所以，一般建議 hooks 為 ```(base autodetect fsck strip)```

### compiling binaries

用 lsinitcpio 或是觀察 hooks 的腳本可以發現，裡面含有的 binaries ，以上方的 hooks 設定為例通常會有 ```busybox``` (來自mkinitcpio-busybox的版本), ```mount```, ```switch_root```, ```blkid```, ```fsck```, ```e2fsck``` ，都是由當前檔案系統複製出來的版本，使用了 glibc ，對於 initramfs 通常 musl 是更好的選擇，大小小很多，效能差異也不大， glibc 特有的功能在 initramfs 通常也都用不到，透過把 binaries 換成用 musl 自行編譯的版本會達到很棒的效果，在自行編譯時，也可以順便針對機子加入 ```-mtune=<cpu-type>``` 或是利用 ```-Os``` ，甚至自行斟酌 static link 或 dynamic link 來進一步減少 binaries 的大小

我採用的策略是使用 musl ，加上 -mtune 再加上 -Os ， libc 採取 dynamic link 其餘採取 static link

musl 在 Arch 的 package 就叫做 ```musl``` ，裡面除了 musl libc 本身，還含有 ```musl-gcc``` 指令( gcc 的 wrapper )，方便使用 musl 編譯，通常在 configure 時把變數 CC 設為 musl-gcc ，或是編譯時到 Makefile 將 CC 設為 musl-gcc 即可

如果需要 musl 的 ldd ，例如在修改 mkinitcpio 找尋需要的 libraries 的部份，或是觀察 linking ，直接執行 /lib/ld-musl-\*.so 即可，也可以把它 link 成 musl-ldd 之類的方便使用

#### [busybox](https://busybox.net/downloads/)

busybox 的 config 方式是採用 Kconfig ，執行 ```make menuconfig``` 就能調整需要的功能

觀察 mkinitcpio 的 init 腳本就能找出所有需要的指令，可以減少到只剩需要的功能，也可以留下一些基本的指令用來除錯

busybox 的 ```mount```, ```switch_root```, ```fsck``` 實做功能已經足夠在 initramfs 使用，可以改用 busybox 的實做減少實際需要的 binaries 數量

其中 busybox 的 mount 能判別出 ext 系列，但沒有分辨是 ext2, ext3 或 ext4 ，會從 ext2 開始試著 mount ，建議 cmdline 加入 rootfstype 直接指定

busybox 對於 ```blkid``` 就比較不全面了，只能查詢 UUID ，建議還是用一般的 blkid

其中比較需要注意的是 long options 的支援， musl 的 getopt 不會找尋 non-option 後方的 option ，但是 getopt_long 會，所以建議要打開，會用到的例子是 init 在 mount 的時候是執行 ```mount -t <type> <dev> <dist> -o <options>``` ，如果用 getopt 會抓不到後面的 -o 項，造成 busybox 檢查 args 的數量時出錯 ( getopt 在這種情況下的表現其實 POSIX 沒有定義到)

我的 [config](https://github.com/xdavidwu/arch-linux-boot-optimize/blob/master/busybox-config)

#### [util-inux](https://git.kernel.org/pub/scm/utils/util-linux/util-linux.git/)

提供 ```blkid```

#### [e2fsprogs](https://git.kernel.org/pub/scm/fs/ext2/e2fsprogs.git/)

提供 ```e2fsck``` ，即 ```fsck.ext4```,```fsck.ext3```, ```fsck.ext2```



## systemd

### services

首先當然是停用不必要的 service ，以 ```sudo systemctl disable``` 停用，列出已經啟用的 service 最簡單的方式就是利用 bash-completion 在 disable 後方按兩次 ```<tab>``` 鍵

再來觀察 ```systemd-analyze blame``` ，不必要且沒有明確 enable 但自動載入的可以用 ```sudo systemctl mask``` 擋掉

### journal flush

如果系統用久了就會發現 ```systemd-journal-flush.service``` 占很大的時間，但如果停用它 journal 就會留在 /run 而不是移到 /var ，關機了就沒了，解決方法是停用他但在其他時機手動 flush ，或是限制 journal 的大小避免花費太多時間，見 ```/etc/systemd/journald.conf```

### remount

如果把 fsck 交由 initramfs 實行， systemd remount 一次 root 就顯得多餘了，確定 cmdline 有 rootflags=rw 等想要的 mount flags 後可以把 /etc/fstab 的 root 註解掉避免 remount

### mandb

現在 Arch 的作法是每隔 12h 開機時更新一次 mandb ，但這通常很慢，一個作法是改回原本的方法，利用 pacman hooks 在有更動到 manpages 時更新，不過這把時間轉嫁給了 pacman ，在 AUR 上有 mandb-ondemand ，同樣是個 pacman hook 但利用 systemd 在背景執行不須等待，安裝它後 mask man-db.service 即可



## quiet boot

解決大部分的東西後， message 也顯得不太重要了， vga console 或 framebuffer console 的效能都不佳，甚至可能成為瓶頸，建議停用一些 message 來加速開機

### kernel console printk

直接在 config 中把 printk 的功能拿掉或許會減少很多空間，但這樣就很難 debug 了，建議留著，讓他只把錯誤訊息輸出在 console 就好

確定 kernel 的 cmdline 有 ```quiet``` 這項，可以透過 ```/proc/cmdline``` 檢查，從 ```/etc/default/grub``` 的 ```GRUB_CMDLINE_LINUX``` 修改

### systemd boot message

如果 cmdline 有了 quiet ， systemd 就只會在出錯時開始輸出，但有時候還是會有不需要又關不掉的功能的報錯，比如說找不到 autofs4 的 module 之類的，這時候還是可以在 cmdline 加入 ```systemd.show_status=false``` 強制關掉

關了這些後，理想狀況下linux的開機輸出就差不多只剩initramfs的fsck了，就把他留著吧
