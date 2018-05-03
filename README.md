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
