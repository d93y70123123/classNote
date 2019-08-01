如果要使用UEFI進行網路開機的話，需要各項硬體配備支援。  
其中最重要的就是網卡的支援了，網卡需有網路堆疊(network stack)的功能。  

下面會分別以虛擬機進行UEFI、BIOS的製作。  
## 先從BIOS開始  
虛擬機建立，配置如下：  
* 一個小小的大約佔用 5G 容量的 Linux，重點在管理開機選單。  
* 大約 100G 的 windows，上課專用的。  
* 大約 30G 的 windows，考試環境專用的。  

Linux只需要根目錄 5G，其他不需要，且以最小安裝即可。  
1. 更新系統，安裝會用到的基本指令。  
2. 開始分割  
    * 第一個Windows 100G，格式化為NTFS。  
    * 第一個Windows 30G ，格式化為NTFS。  



## 接下來就可以安裝第一個Windows  
*需要有的檔案:win10.iso、virtio

透過虛擬機安裝需要切換驅動還有win10iso檔。
驅動是透過virtio安裝:  
* amd64/win10 --> Red Hat VirtIO SCSI controller
* NetKVM/win10/amd64 --> Red Hat VirtIO Ethernet Adapter
* qemufwcfg/win10/amd64 --> QEMU FWCfg Device
* qxldod/win10/amd64 --> Red Hat QXL controller
* viorng/win10/amd64 --> VirtIO RNG Device  

安裝以上幾個即可。  
透過virsh attach-disk置換光碟
```bash
# virsh attach-disk bios /vmdisk/iso/virtio-win-0.1.171.iso hda --type cdrom --mode readonly
Disk attached successfully
```

## 建第二個Windows
可以先進入rootfs，將第一個windows(partition2)隱藏。
```
# fdisk -l /dev/vda
Disk /dev/vda: 145.0 GB, 144955146240 bytes, 283115520 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O 大小 (最小/最佳化)：512 位元組 / 512 位元組
Disk label type: dos
磁碟識別碼：0x000c07c2

所用裝置 開機      開始         結束      區塊   識別號  系統
/dev/vda1            2048    10487807     5242880   83  Linux
/dev/vda2        10487808   220203007   104857600   93  Amoeba           <==這裡要變成這樣！
/dev/vda3       220203008   283115519    31456256    7  HPFS/NTFS/exFAT
```

隱藏後就可以重複上個步驟，安裝第二個Windows  

## 開機選單救援
當Windows兩個都安裝完之後，就會發現進不去之前的系統。  
所以接下來要把開機選單救回來。  
進入rootfs之後要先把系統掛載回來：
```
mkdir /sysroot
mount /dev/vda1 /sysroot
mount --bind /proc /sysroot/proc
mount --bind /sys /sysroot/sys
mount --bind /dev /sysroot/dev
chroot /sysroot/
```

檢查下有沒有把系統都打開  
```
# fdisk -l /dev/vda
Disk /dev/vda: 145.0 GB, 144955146240 bytes, 283115520 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O 大小 (最小/最佳化)：512 位元組 / 512 位元組
Disk label type: dos
磁碟識別碼：0x000c07c2

所用裝置 開機      開始         結束      區塊   識別號  系統
/dev/vda1            2048    10487807     5242880   83  Linux
/dev/vda2        10487808   220203007   104857600    7  HPFS/NTFS/exFAT
/dev/vda3       220203008   283115519    31456256    7  HPFS/NTFS/exFAT
```

將剛剛的windows系統做成開機選單
```bash
#!/bin/sh
exec tail -n +3 $0
# This file provides an easy way to add custom menu entries.  Simply type the
# menu entries you want to add after this comment.  Be careful not to change
# the 'exec tail' line above.
menuentry "Windows 10 - Class" {
        insmod chain
        insmod ntfs
        parttool hd0,msdos2 hidden- boot+
        parttool hd0,msdos3 hidden+ boot-
        set root='hd0,msdos2'
        chainloader +1
}
menuentry "Windows 10 - Exam" {
        insmod chain
        insmod ntfs
        parttool hd0,msdos2 hidden+ boot-
        parttool hd0,msdos3 hidden- boot+
        set root='hd0,msdos3'
        chainloader +1
}
```

重新建立開機選單，然後將選單安裝在第一個磁區(/dev/vda)裡。
```
# grub2-mkconfig -o /boot/grub2/grub.cfg
# grub2-install /dev/vda
# poweroff
```

重新開機然後檢查各個作業系統有沒有問題。  

## grub4dos 專門為Windows製作的開機選單
```
# 1. 先安裝好 p7zip 這個支援的壓縮軟體：
# yum --enablerepo=epel install p7zip

# 開始下載 grub4dos
# mkdir /vmdisk/rootfs/grub4dos
# cd /vmdisk/rootfs/grub4dos
# wget http://dl.grub4dos.chenall.net/grub4dos-0.4.6a-2019-05-12.7z

# 2. 開始在 /var/www/html/ipxe 底下進行解壓縮：
# mkdir /var/www/html/ipxe/grub
# cd /var/www/html/ipxe/grub
# 7za  e  /vmdisk/rootfs/grub4dos/grub4dos-0.4.6a-2019-05-12.7z
```

修改一下ipxe的選單
1. 者的差異是這樣的：
Linux / Grub4dos / Grub2
(/dev/vda) / (hd0) / (hd0)
(/dev/vda1) / (hd0,0) / (hd0,1)
(/dev/vda2) / (hd0,1) / (hd0,2)
現在，就讓我們來做個簡單的設定吧！  
```
# 1. 處理一下 /var/www/html/ipxe/menu.php
# vim /var/www/html/ipxe/menu.php
...
:start
menu iPXE ${version} Boot Menu (ip ${net0/ip}, mac ${net0/mac})
item --gap -- ------------------------- Select your OS----------------------------------
item diskless             Go To diskless
item local                Go To Local HD booting
item win10class           Go To Windows 10 Class
item win10exam            Go To Windows 10 Exam
...

:win10class
echo Go To Windows 10 class
set base-url http://${net0/gateway}/ipxe/grub
chain ${base-url}/grub.exe --config-file=" hide (hd0,2); unhide (hd0,1); rootnoverify (hd0,1); chainloader +1; makeactive" || goto failed

:win10exam
echo Go To Windows 10 Exam
set base-url http://${net0/gateway}/ipxe/grub
chain ${base-url}/grub.exe --config-file=" hide (hd0,1); unhide (hd0,2); rootnoverify (hd0,2); chainloader +1; makeactive" || goto failed
```
