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

如果只有一個系統就直接把timeout拿掉吧

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



## systemd

### services

首先當然是停用不必要的service，以```sudo systemctl disable```停用，列出已經啟用的service最簡單的方式就是利用bash-completion在disable後方按兩次```<tab>```鍵

再來觀察```systemd-analyze blame```，不必要且沒有明確enable但自動載入的可以用```sudo systemctl mask```擋掉

### journal flush

如果系統用久了就會發現```systemd-journal-flush.service```占很大的時間，但如果停用它journal就會留在/run而不是移到/var，關機了就沒了，解決方法是停用他但在其他時機手動flush，或是限制journal的大小避免花費太多時間，見```/etc/systemd/journald.conf```

### remount

如果把fsck交由initramfs實行，systemd remount一次root就顯得多餘了，確定cmdline有rootflags=rw等想要的mount flags後可以把/etc/fstab的root註解掉避免remount



## quiet boot

解決大部分的東西後，message也顯得不太重要了，vga console或framebuffer console的效能都不佳，甚至可能成為瓶頸，建議停用一些message來加速開機

### kernel console printk

直接在config中把printk的功能拿掉或許會減少很多空間，但這樣就很難debug了，建議留著，讓他只把錯誤訊息輸出在console就好

確定kernel的cmdline有```quiet```這項，可以透過```/proc/cmdline```檢查，從```/etc/default/grub```的```GRUB_CMDLINE_LINUX```修改

### systemd boot message

如果cmdline有了quiet，systemd就只會在出錯時開始輸出，但有時候還是會有不需要又關不掉的功能的報錯，比如說找不到autofs4的module之類的，這時候還是可以在cmdline加入```systemd.show_status=false```強制關掉



關了這些後，理想狀況下linux的開機輸出就差不多只剩initramfs的fsck了，就把他留著吧
