## 上次用BIOS開機，這次要用UEFI取代BIOS開機。
*補充一下：MBR跟GPT的最明顯的差異在於，MBR只能支援到2TB，BIOS依舊存在只能使用 16 位元模式、最大定址為1MB、找到的磁碟容量在 windows 底下最大僅能支援到 2.1T  

首先先查看一下虛擬機有沒有支援q35的型號
```
# /usr/libexec/qemu-kvm -machine help
Supported machines are:
pc                   RHEL 7.6.0 PC (i440FX + PIIX, 1996) (alias of pc-i440fx-rhel7.6.0)
pc-i440fx-rhel7.6.0  RHEL 7.6.0 PC (i440FX + PIIX, 1996) (default)
pc-i440fx-rhel7.5.0  RHEL 7.5.0 PC (i440FX + PIIX, 1996)
pc-i440fx-rhel7.4.0  RHEL 7.4.0 PC (i440FX + PIIX, 1996)
pc-i440fx-rhel7.3.0  RHEL 7.3.0 PC (i440FX + PIIX, 1996)
pc-i440fx-rhel7.2.0  RHEL 7.2.0 PC (i440FX + PIIX, 1996)
pc-i440fx-rhel7.1.0  RHEL 7.1.0 PC (i440FX + PIIX, 1996)
pc-i440fx-rhel7.0.0  RHEL 7.0.0 PC (i440FX + PIIX, 1996)
rhel6.6.0            RHEL 6.6.0 PC
rhel6.5.0            RHEL 6.5.0 PC
rhel6.4.0            RHEL 6.4.0 PC
rhel6.3.0            RHEL 6.3.0 PC
rhel6.2.0            RHEL 6.2.0 PC
rhel6.1.0            RHEL 6.1.0 PC
rhel6.0.0            RHEL 6.0.0 PC
q35                  RHEL-7.6.0 PC (Q35 + ICH9, 2009) (alias of pc-q35-rhel7.6.0)
pc-q35-rhel7.6.0     RHEL-7.6.0 PC (Q35 + ICH9, 2009)
pc-q35-rhel7.5.0     RHEL-7.5.0 PC (Q35 + ICH9, 2009)
pc-q35-rhel7.4.0     RHEL-7.4.0 PC (Q35 + ICH9, 2009)
pc-q35-rhel7.3.0     RHEL-7.3.0 PC (Q35 + ICH9, 2009)
none                 empty machine
```

如果沒有支援q35的話可以檢查一下qemu的版本，可以嘗試升級。
```
yum install centos-release-qemu-ev.noarch
yum --enablerepo=centos-qemu-ev install qemu-kvm-ev
```

主機支援了之後，就是BIOS韌體的部分了，虛擬機預設是使用seabios，所以我們現在要把它改成UEFI的韌體
```bash
# yum install OVMF*

# rpm -qa |grep OVMF
OVMF-20180508-3.gitee3198e672e2.el7_6.1.noarch
```
現在，機器的硬體支援也有了，韌體上的UEFI也有了。

## 現在要針對虛擬機的設定做調整
```xml
<os>
    <type arch="x86_64" machine="q35">hvm</type>
    <loader readonly='yes' type='pflash'>/usr/share/OVMF/OVMF_CODE.secboot.fd</loader>
    <nvram>/vmdisk/uefios.fd</nvram>
    <bootmenu enable='no'/>
    <boot dev="cdrom"/>
    <boot dev="hd"/>
</os>
..
..
<disk type="file" device="cdrom">
      <driver name="qemu" type="raw"/>
      <source file="/vmdisk/iso/CentOS-7-x86_64-DVD-1810.iso"/>
      <target dev="hda" bus="sata"/>
      <readonly/>
 </disk>
```
*為 q35 裝置已經不支援 IDE 的設備，因此要將 IDE 修改成為 sata 裝置才行！  

## 啟動安裝第一套Linux
照著安裝後的基本步驟做
```
# yum -y update
# yum install net-tools bash-completion vim-enhanced tcpdump bind-utils bridge-utils iptables-services
# fdisk /dev/vda
# 依舊需要先切出來額外兩塊分割槽才行：

# fdisk -l /dev/vda
WARNING: fdisk GPT support is currently new, and therefore in an experimental phase. Use at your own discretion.

Disk /dev/vda: 145.0 GB, 144955146240 bytes, 283115520 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O 大小 (最小/最佳化)：512 位元組 / 512 位元組
Disk label type: gpt
Disk identifier: 079FCA41-C1CD-4CC9-9778-D69D901083C7


#         Start          End    Size  Type            Name
 1         2048       411647    200M  EFI System      EFI System Partition
 2       411648     10897407      5G  Linux filesyste
 3     10897408    220612607    100G  Microsoft basic <==用 systemid 為 11 喔！
 4    220612608    283115486   29.8G  Microsoft basic
```
可以用檢查 tail /var/lib/dhcpd/dhcpd.leases 虛擬機取得的IP  

## 安裝第一套Windows
```
# 1. 先放讓 SATA 界面的光碟機，抽換 Windows 10 的安裝光碟片：
# virsh attach-disk uefios /vmdisk/iso/Win10_1809_64.iso sda --targetbus sata --type cdrom --mode readonly

# 2. 換完之後，查看一下是否是正確的光碟資料呢？
# virsh qemu-monitor-command --hmp uefios --cmd "info block"
pflash0 (#block167): /usr/share/OVMF/OVMF_CODE.secboot.fd (raw, read-only)
    Attached to:      /machine/unattached/device[10]
    Cache mode:       writeback

pflash1 (#block302): /vmdisk/uefios.fd (raw)
    Attached to:      /machine/unattached/device[11]
    Cache mode:       writeback

drive-virtio-disk0 (#block516): /vmdisk/uefios.img (qcow2)
    Attached to:      /machine/peripheral/virtio-disk0/virtio-backend
    Cache mode:       writeback

drive-sata0-0-0 (#block1325): /vmdisk/iso/Win10_1809_64.iso (raw, read-only)
    Attached to:      sata0-0-0
    Removable device: not locked, tray closed
    Cache mode:       writeback
```
利用置換光碟的功能，切換驅動跟印象檔來安裝。
其他就跟之前BIOS安裝Windows的步驟一樣。

## 安裝第二套Windows
UEFI不能隱藏作業系統
```
# 1. 先重新開機進入 CentOS 的環境：
# 按照剛剛的流程， 『Boot Manager』 --> 『CentOS』來啟動吧！

# 2. 進入 Linux 之後，可以使用 ssh 進去系統工作即可：
# df -T | grep -v 'tmpfs'
檔案系統       類型     1K-區段    已用    可用 已用% 掛載點
/dev/vda2      ext4     5029504 1765900 2985076   38% /
/dev/vda1      vfat      204580   36648  167932   18% /boot/efi  <==就是這裡囉！

# 3. 開始查看這個 UEFI 的開機選單紀錄資料裡面有什麼鬼？
# cd /boot/efi
# find . -type d
./EFI
./EFI/centos                   <==CentOS 開機選單紀錄
./EFI/centos/fonts
./EFI/BOOT                     <==UEFI 紀錄
./EFI/Microsoft                <==Microsoft 自己的選單紀錄
./EFI/Microsoft/Boot           <==開機資料
./EFI/Microsoft/Boot/bg-BG     <==各國語系界面資料
./EFI/Microsoft/Boot/cs-CZ
.....
./EFI/Microsoft/Boot/zh-CN
./EFI/Microsoft/Boot/zh-TW
./EFI/Microsoft/Boot/Fonts
./EFI/Microsoft/Boot/Resources <==其他資源
./EFI/Microsoft/Boot/Resources/en-US
./EFI/Microsoft/Boot/Resources/zh-TW
./EFI/Microsoft/Recovery       <==系統救援方式

# 4. 確實開始更改 Microsoft 的開機選單位置，改成 win10class 好了：
# cd /boot/efi/EFI/
# ll
drwx------. 2 root root 4096  5月 31 11:44 BOOT
drwx------. 3 root root 4096  5月 31 21:54 centos
drwx------. 4 root root 4096  5月 31 22:04 Microsoft  <==將這個改掉！

# mv Microsoft win10class
# ll win10class/Boot/bootmgfw.efi
-rwx------. 1 root root 1469752  9月 15  2018 win10class/Boot/bootmgfw.efi
# 上面這個檔案，就是 windows 的開機引導程式囉！
```

## 處理選單
現在的選單管理位置在 /boot/efi/EFI/
製作選單
```
# 1. 先處理 40_custom 的檔案內容：
# vim /etc/grub.d/40_custom
#!/bin/sh
exec tail -n +3 $0
# This file provides an easy way to add custom menu entries.  Simply type the
# menu entries you want to add after this comment.  Be careful not to change
# the 'exec tail' line above.
menuentry "Windows 10 Class system" {
        insmod part_gpt
        insmod search_fs_uuid
        insmod chain
        set root=(hd0,gpt1)
        chainloader /EFI/win10class/Boot/bootmgfw.efi
}
menuentry "Windows 10 Exam environment" {
        insmod part_gpt
        insmod search_fs_uuid
        insmod chain
        set root=(hd0,gpt1)
        chainloader /EFI/win10exam/Boot/bootmgfw.efi
}
```

選單安裝
grub2-mkconfig -o /boot/efi/EFI/centos/grub.cfg

## UEFI的IPXE

## rootfs搭配partclone

