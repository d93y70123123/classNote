## 簡單描述開機流程
Linux:  
按下電源 >> BIOS/UEFI >> MBR / efibooting >> kernel+initrd >> systemd 呼叫根目錄  
通常開機之後，都會先看到廠牌LOGO畫面，接著跑到windows的標誌然後登入。
看到LOGO的時候就是在跑MBR等開機引導程式，然後核心跟驅動(版本必須一樣)被呼叫，接著就進入作業系統的開機程序。  

## 無碟環境
透過網路抓到核心跟驅動程式達到沒有硬碟就可以開機的需求。
1. 必須要透過DHCP抓到網路，才能獲得IP。
2. 下載開機管理程式
3. 取得開機管理程式後，就可以下載核心。
4. 接下來才能執行核心等參數

## rootfs
準備根目錄，利用先前建立的虛擬機複製環境。  
```
需要安裝 libguestfs-tools-c
virt-copy-out -a /vmdisk/demo.img / /vmdisk/rootfs/filesystem
* 在img後面有根目錄要注意，把印象檔裡的根目錄複製到實體主機的目錄
# du -sm *
0       bin
100     boot  <==好像大了點
0       dev
35      etc
0       home
0       lib
0       lib64
0       media
0       mnt
0       opt
0       proc
1       root
0       run
0       sbin
0       srv
0       sys
1       tmp
1068    usr   <==好像大了點
93      var
```

幫根目錄塑身。
```
# 1. 先檢查 boot/ 底下有什麼東西？
# ll boot/
-rw-r--r--. 1 root root   151923  3月 18 23:10 config-3.10.0-957.10.1.el7.x86_64
drwxr-xr-x. 3 root root       17  3月 12 19:46 efi
drwxr-xr-x. 2 root root       27  3月 12 19:47 grub
drwx------. 5 root root       97  4月 23 10:56 grub2
-rw-------. 1 root root 57153862  3月 12 19:51 initramfs-0-rescue-fcae882ccb0f4ccc9e6b3461dd810057.img
-rw-------. 1 root root 21318524  3月 19 19:17 initramfs-3.10.0-957.10.1.el7.x86_64.img
-rw-r--r--. 1 root root   314087  3月 18 23:10 symvers-3.10.0-957.10.1.el7.x86_64.gz
-rw-------. 1 root root  3544363  3月 18 23:10 System.map-3.10.0-957.10.1.el7.x86_64
-rwxr-xr-x. 1 root root  6639904  3月 12 19:51 vmlinuz-0-rescue-fcae882ccb0f4ccc9e6b3461dd810057
-rwxr-xr-x. 1 root root  6643904  3月 18 23:10 vmlinuz-3.10.0-957.10.1.el7.x86_64

# 2. 看起來跟 rescue 有關的那兩個項目，就直接刪除吧！不需要保留了！
# rm boot/*rescue*

# 3. 因為還需要某些軟體，所以直接來安裝一下這些軟體：
# yum --installroot /vmdisk/rootfs/filesystem/ install \
> bash-completion vim-enhanced lsof epel-release net-tools pciutils wget tcpdump

# 3.1 某些軟體是 EPEL 提供的，所以要分兩次來安裝！
# yum --installroot /vmdisk/rootfs/filesystem/ install \
> ntfs-utils ntfs-3g ntfsprogs partclone

# 4. 基礎系統的設定處理：
# vim etc/selinux/config
SELINUX=disabled

# vim etc/ssh/sshd_config
UseDNS no
MaxSessions 100

# rm dev/null

# vim etc/fstab
這個檔案清空！不需要留下任何有效的資訊！

# vim etc/yum.repos.d/epel.repo
enabled=0  # 所有的 enable 都設定為 0 喔！

# 5. 開始移除暫時可能用不到的軟體！
# yum --installroot /vmdisk/rootfs/filesystem/ remove \
   selinux-policy-targeted-3.13.1-229.el7_6.9.noarch \
   libselinux-utils-2.5-14.1.el7.x86_64 \
   container-selinux-2.74-1.el7.noarch  \
   NetworkManager-* \
   firewall*  container* 

# 6. 檢查一下該目錄下所有的軟體名稱：
# rpm --root /vmdisk/rootfs/filesystem/ -qa | sort

# 7. 通常電腦教室的主機是不需要 wifi 模組的，所以，可以將 iwl??-firmware 的軟體移除
# yum --installroot /vmdisk/rootfs/filesystem/ remove iwl*-firmware

# 8. 將剛剛使用 yum 所產生的所有暫存資訊都移除
# yum --installroot /vmdisk/rootfs/filesystem/ --enablerepo=epel clean all

# du -sm
0       bin
39      boot
1       dev
12      etc
0       home
0       lib
0       lib64
0       media
0       mnt
0       opt
0       proc
1       root
0       run
0       sbin
0       srv
0       sys
1       tmp
890     usr
93      var

```

## 建立initramfs
目錄規劃
```
/vmdisk/rootfs/filesystem/		# 由 VM image 裡面取得的根目錄
/vmdisk/rootfs/filesystem.d/		# 預計放置由上述目錄壓縮來的根目錄 tar 包
/vmdisk/rootfs/initramfs/		# 就是 initramfs 的執行流程所需要的各項資料
```

編譯busybox  
這邊可以下載 : https://busybox.net/downloads/?C=M;O=D  
注意不要使用 1.28.2 的預編譯版本
```
# 1. 先讓我們的系統可以進行編譯，因此先安裝開發工具：
# yum groupinstall "Development Tools"
# yum install glibc-static

# 2. 開始下載 busybox，我們在 2019/05/21 下載的最新版本為 1.30.1 這個穩定版：
# cd /vmdisk/rootfs
# mkdir busybox
# cd busybox
# wget https://busybox.net/downloads/busybox-1.30.1.tar.bz2

# 3. 解壓縮之後就開始來編譯了！
# tar -jxvf busybox-1.30.1.tar.bz2
# cd busybox-1.30.1
# make defconfig
# LDFLAGS="--static" make -j 4
...這會花去一小段時間...

# 4. 觀察編譯完成的檔案內容喔
# ll busybox
-rwxr-xr-x. 1 root root 2651704  5月 21 15:27 busybox

# file busybox
busybox: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, 
  for GNU/Linux 2.6.32, BuildID[sha1]=a8013757b964f2920c0b7e0e63979862bc0c09c9, stripped
```

準備建立initrd囉  
```
# 1. 先建立所需要的目錄：
# cd /vmdisk/rootfs
# mkdir filesystem.d initramfs
# cd initramfs

# 2. 將剛剛編譯完成的 busybox 複製到 bin 底下喔：
# mkdir bin
# cp ../busybox/busybox-1.30.1/busybox bin/busybox
# chmod a+x bin/busybox
# ll bin
-rwxr-xr-x. 1 root root 2651704  5月 21 15:28 bin/busybox

# 3. 建立 init 這個執行腳本！
# vim init
#!/bin/busybox sh

# 如果有任何執行過程發生錯誤，就丟一個 busybox 的 shell 給管理員 debug 之用！
error() {
        echo "Jumping into the shell..."
        setsid cttyhack sh
}

# 開始執行 busybox 內的各項程序
/bin/busybox --install /bin

# 將核心運作過程中，偵測到的 proc 與 sys 掛載到正確的目錄去
mkdir -p /proc
mount -t proc proc /proc

mkdir -p /sys
mount -t sysfs sysfs /sys

# 將核心偵測到的硬體裝置，一項一項的掛載到正確的目錄去
mkdir -p /sys/dev
mkdir -p /var/run
mkdir -p /dev

mkdir -p /dev/pts
mount -t devpts devpts /dev/pts

# 處理裝置檔案，加入熱拔插裝置 /dev
echo /bin/mdev > /proc/sys/kernel/hotplug
mdev -s

# 建立 tmpfs 的載點，並提供適當的記憶體成為根目錄，1500m 就跟你的記憶體有關！
mkdir -p /newroot
mount -t tmpfs -o size=1500m tmpfs /newroot || error

# 開始針對我們建立的根目錄檔案系統解壓縮
echo "Extracting rootfs... "
xz -d -c -f rootfs.tar.xz | tar -x -f - -C /newroot || error

# 將核心偵測到的 /sys, /proc, /dev 整個移動到下一個跟目錄環境去
mount --move /sys /newroot/sys
mount --move /proc /newroot/proc
mount --move /dev /newroot/dev

# 最終就是執行未來根目錄中的 systemd
exec switch_root /newroot /usr/lib/systemd/systemd || error

# chmod a+x init

# 4. 將 init 會運作到的各項資料進行壓縮打包的任務：
# cd /vmdisk/rootfs
# vim create.initramfs.sh
#!/bin/bash

# 1. 先設定好會用到的目錄所在
basedir=/vmdisk/rootfs
rootdir=${basedir}/filesystem
rootfsdir=${basedir}/filesystem.d
date=$( date +%Y%m%d )
ramdir=${basedir}/initramfs/
rootfsfile=${rootfsdir}/rootfs.tar.xz-${date}
ramfs=${basedir}/initramfs.gz-${date}

if [ ! -d ${rootdir} -o ! -d ${rootfsdir} -o ! -d ${ramdir} ]; then
        echo "Important directory NOT exist"
        exit 1
fi

# 2. 預防自己忘記加上 yum 的清除，所以這裡再次清理一次 yum 佔存檔
echo "1. clean yum log file"
yum --installroot=${rootdir} clean all

# 3. 因為 xz 提供壓縮多執行緒，所以這裡改成這樣，可以加快壓縮的效能！
echo "2. start to create rootfs"
cd ${rootdir}
time tar -c -f - . |  xz -T 0 -9 -c - > ${rootfsfile}
time cp -a ${rootfsfile} ${ramdir}/rootfs.tar.xz

# 4. 最終，因為核心使用的是 cpio 這個指令來進行檔案的處理，所以就這樣進行即可！
echo "3. create initramfs"
cd ${ramdir}
time find . -print0 | cpio --null -o -H newc --quiet | gzip > ${ramfs}

# chmod a+x create.initramfs.sh
# ll
-rwxr-xr-x.  1 root root 1047  5月 20 11:05 create.initramfs.sh
dr-xr-xr-x. 17 root root  224  4月 23 11:34 filesystem
drwxr-xr-x.  2 root root    6  5月 20 10:45 filesystem.d
drwxr-xr-x.  3 root root   29  5月 20 10:57 initramfs

# 5. 最終就是運作他囉！ 
# ./create.initramfs.sh
```

## 建立 iPXE 的網路開機環境
```
# 1. 先處理想要編譯 iPXE 時，所需要的基礎環境：
# yum groupinstall "Development Tools"
# yum install xz-devel

# 2. 使用 git 來下載所需要的 iPXE 原始碼：
# cd /vmdisk/rootfs/
# git clone git://git.ipxe.org/ipxe.git
# cd ipxe
# ll
drwxr-xr-x.  6 root root    78  5月 20 11:49 contrib
-rw-r--r--.  1 root root   558  5月 20 11:49 COPYING
-rw-r--r--.  1 root root 18092  5月 20 11:49 COPYING.GPLv2
-rw-r--r--.  1 root root  2931  5月 20 11:49 COPYING.UBDL
-rw-r--r--.  1 root root   113  5月 20 11:49 README
drwxr-xr-x. 19 root root  4096  5月 20 11:49 src

# 3. 建立 chainloader 所需要的 iPXE 核心檔案：
# cd src
# vim Makefile  # 自己看看，查詢一下 undionly.kpxe 這個關鍵字
# make bin-x86_64-pcbios/undionly.kpxe
... (會花費一小段時間)

# ll bin-x86_64-pcbios/undionly.kpxe
-rw-r--r--. 1 root root 75468  5月 20 12:00 bin-x86_64-pcbios/undionly.kpxe
```

建立 tftp 服務
```
# yum install tftp tftp-server
# systemctl start tftp.socket
# systemctl enable tftp.socket
```

開始搬移 undinoly.kpxe 檔案到正確的位置：
```
# cd /vmdisk/rootfs/ipxe/src/bin-x86_64-pcbios/
# cp undionly.kpxe /var/lib/tftpboot/
```

開始處理 dhcp 服務的設定：
```
# 1. 開始修改 dhcpd.conf
# vim /etc/dhcp/dhcpd.conf
default-lease-time 600;
max-lease-time 72000;
ddns-update-style none;
log-facility local7;

subnet 192.168.19.0 netmask 255.255.255.0 {
  range 192.168.19.1 192.168.19.150;
  option routers 192.168.19.254;
  option domain-name "virtual.dic";
  option domain-name-servers 120.114.100.1,120.114.150.1;

  next-server 192.168.19.254;
  if exists user-class and option user-class = "iPXE" {
          filename "http://192.168.19.254/ipxe/menu.php";
  } else {
          filename "undionly.kpxe";
  }

  host fantasia {
    hardware ethernet 52:54:00:84:c7:52;
    fixed-address 192.168.19.9;
  }
}

# 2. 重新啟動 dhcp 吧！
# systemctl restart dhcpd
# systemctl status dhcpd
```

開始透過 web 提供開機選單的任務：就是 /var/www/html/ipxe/menu.php 的內容！
```
# 1. 安裝軟體
# yum install httpd php
# systemctl restart httpd

# 2. 開始設計 menu.php 的內容：
# cd /var/www/html
# mkdir ipxe
# cd ipxe
# vim menu.php
#!ipxe

set menu-timeout 60000
set submenu-timeout ${menu-timeout}
isset ${menu-default} || set menu-default diskless

:start
menu iPXE ${version} Boot Menu (ip ${net0/ip}, mac ${net0/mac})
item --gap -- ------------------------- Select your OS----------------------------------
item diskless             Go To diskless 
item local                Go To Local HD booting
item --gap --
item --gap -- ------------------------- Advanced Options -------------------------------
item memtest              Go To Memtest86
item shell                Drop to iPXE shell
item instcent             Install CentOS 7 use KSU FTP
item reboot               Reboot computer
choose --timeout ${menu-timeout} --default ${menu-default} selected || goto exit
set menu-timeout 0
goto ${selected}

:diskless
echo Boot from rootfs diskless
kernel vmlinuz devfs=nomount rw panic=60 selinux=0 ip=dhcp biosdevname=0 net.ifnames=0 ipv6.disable=0
initrd initramfs.gz
boot || goto failed
goto start

:local
sanboot --no-describe --drive 0x80

:memtest
set base-url http://${net0/gateway}/ipxe
kernel ${base-url}/memdisk || read void
initrd ${base-url}/memtest86+-5.01.iso || read void
imgargs memdisk iso raw || read void
boot || goto failed

:shell
echo Type 'exit' to get the back to the menu
shell
set menu-timeout 0
set submenu-timeout 0
goto start

:instcent
set base-url http://${net0/gateway}/centos7
kernel ${base-url}/images/pxeboot/vmlinuz initrd=initrd.img repo=${base-url}
initrd ${base-url}/images/pxeboot/initrd.img
boot || goto failed

:reboot
reboot

:failed
echo Booting failed, dropping to shell
goto shell

:exit
sanboot --no-describe --drive 0x80
```

把核心跟initramfs放到網頁設定的位置上
```
# cp /vmdisk/rootfs/filesystem/boot/vmlinuz-3.10.0-957.10.1.el7.x86_64 /var/www/html/ipxe/
# cp /vmdisk/rootfs/initramfs.gz-20190521 /var/www/html/ipxe/
# cd /var/www/html/ipxe
# ln -s initramfs.gz-20190521 initramfs.gz
# ln -s vmlinuz-3.10.0-957.10.1.el7.x86_64 vmlinuz
# ll
lrwxrwxrwx. 1 root root        21  5月 21 14:14 initramfs.gz -> initramfs.gz-20190521
-rw-r--r--. 1 root root 263825132  5月 21 15:52 initramfs.gz-20190521
-rw-r--r--. 1 root root      1892  5月 21 14:23 menu.php
lrwxrwxrwx. 1 root root        34  5月 21 14:15 vmlinuz -> vmlinuz-3.10.0-957.10.1.el7.x86_64
-rwxr-xr-x. 1 root root   6643904  5月 21 14:09 vmlinuz-3.10.0-957.10.1.el7.x86_64
```

接下來就可以開啟虛擬機測試了，記得從XML設置網路優先開機
```xml
<boot dev='network'/>
<boot dev="cdrom"/>
<boot dev="hd"/>
```
