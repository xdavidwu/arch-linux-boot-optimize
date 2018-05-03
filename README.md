# Arch Linux 開機加速

## 說明

雖然是在Arch Linux上實驗的但想法都能套用在其他發行版上

## 要點

減少kernel和initramfs的大小、刪減不必要的操作

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

```loader```有包含從硬碟載入kernel和initramfs的時間，這部分影響最大

如果有多個entry，把目標設為預設，開機時壓著方向右或是enter就能消除人為的測量誤差

如果只有一個系統就直接把timeout拿掉吧

如果不在意外觀也可以不用圖形化的介面

再來影響小很多(但通常還是測的出來)的就是減少載入的mod

在UEFI下預設會載入GPT和MBR兩種table的mod，如果不需要用MBR就在```/etc/default/grub```把他拿掉

顯示部分的mod在UEFI底下預設會載入```video.lst```列出的四種：

* ```efi_uga```: EFI時代的顯示方式

* ```efi_gop```: UEFI大多是這個

* ```video_bochs```: bochs模擬器

* ```video_cirrus```: qemu常用的video

用```/etc/default/grub```的```GRUB_VIDEO_BACKEND```可以指定，如果有就只會載入指定的

修改```/etc/default/grub```後都需要手動```sudo grub-mkconfig -o /boot/grub/grub.cfg```才會套用，詳見```/etc/grub.d/00_header```

font觀察```/boot/grub/grub.cfg```，預設是載入Arch Linux的/usr裡面的，但可以發現另一個是直接```loadfont unicode```，直接搜尋```/boot/grub/font```裡面的，修改成這種方式可以跳過載入額外的分區

## Linux kernel

## initramfs

## systemd

首先當然是停用不必要的service，以```sudo systemctl disable```停用，列出已經啟用的service最簡單的方式就是利用bash-completion在disable後按兩次tab鍵

再來觀察```systemd-analyze blame```，不必要且沒有明確enable但自動載入的可以用```sudo systemctl mask```擋掉

如果系統用久了就會發現```systemd-journal-flush.service```占很大的時間，但如果停用它journal就會留在/run而不是移到/var，關機了就沒了，解決方法是停用他但在其他時間手動flush，或是限制journal的大小避免花費太多時間，見```/etc/systemd/journald.conf```
