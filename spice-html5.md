# Use spice-html5 on centos7 
如果要遠端虛擬機，通常都會使用remote-viewer等軟體，但如果可以用網頁來遠端不是很方便嗎  

## 軟體需求  
1. spice-html5  
2. websockify  
沒錯，只需要這兩個軟體就可以達成目標了。第一個軟體應該很好理解，我們就是要使用它麻~
那第二個是甚麼呢?  當我們想要使用spice-html5就需要一個websocket，而websockify就是可以用來代替它且讓我們正常使用的好工具  

## 安裝  
**安裝spice-html5**  
```bash
[root@dic ~]# yum install --enablerepo=epel spice-html5
Loaded plugins: fastestmirror, langpacks
Loading mirror speeds from cached hostfile
epel/x86_64/metalink                                                                                                        | 7.6 kB  00:00:00
 * base: ftp.ksu.edu.tw
 * centos-qemu-ev: ftp.ksu.edu.tw
 * epel: my.mirrors.thegigabit.com
 * extras: ftp.ksu.edu.tw
 * updates: ftp.ksu.edu.tw
base                                                                                                                        | 3.6 kB  00:00:00
centos-qemu-ev                                                                                                              | 2.9 kB  00:00:00
epel                                                                                                                        | 5.3 kB  00:00:00
extras                                                                                                                      | 2.9 kB  00:00:00
updates                                                                                                                     | 2.9 kB  00:00:00
(1/2): epel/x86_64/updateinfo                                                                                               | 1.0 MB  00:00:01
(2/2): epel/x86_64/primary_db                                                                                               | 6.9 MB  00:00:16
Package spice-html5-0.1.7-1.el7.noarch already installed and latest version
Nothing to do
```

**安裝websockify**  
先從github上抓下來
網址：https://github.com/novnc/websockify
```bash
[root@dic ~]# git clone https://github.com/novnc/websockify.git

[root@dic ~]# ll /root/w* -d
drwxr-xr-x. 10 root root 4096 12月 11 01:29 /root/websockify
```
用setup.py安裝，這邊官網有建議如果不需要用到numpy的話可以編輯setup.py並把install_requires=['numpy']刪除  

安裝時可能會遇到python版本問題，會建議要在python3.5以上  
除了安裝python也可以安裝pip這個工具，當然不裝也是ok的  
```bash
[root@dic ~]# yum install --enablerepo=epel python36
[root@dic ~]# yum install -y python3-pip
```

接著安裝websockify
```bash
[root@dic ~]# cd websockify
[root@dic websockify]# pwd
/root/websockify
[root@dic websockify]# python3 setup.py install

[root@120-114-142-26 websockify]# python3 -m pip list
Package    Version
---------- -------
pip        19.3.1
setuptools 39.2.0
websockify 0.9.0
```

## 使用spice-html5  

先確認安裝的spice-html5，把spice.html移動到apache的預設目錄底下
```bash
[root@dic ~]# rpm -qa |grep spice-html5
spice-html5-0.1.7-1.el7.noarch
[root@dic ~]# rpm -ql spice-html5
/usr/share/doc/spice-html5-0.1.7
/usr/share/doc/spice-html5-0.1.7/COPYING
/usr/share/doc/spice-html5-0.1.7/COPYING.LESSER
/usr/share/doc/spice-html5-0.1.7/README
...
/usr/share/spice-html5/spice.html
....

[root@dic ~]# cp -r /usr/share/spice-html5/ /var/www/html/
```

我們要透過websockify讓外面的可以瀏覽到我們的畫面，所以server必須要先開起port才能夠連線且有畫面  
**開啟websockify**  
```bash
[root@dic ~]# websockify/websockify.py 5940 localhost:5900
```
上面這行指令的意思是，將瀏覽到5940的port導到5900，5900就是虛擬機開啟的port，所以囉~自己做替換吧  

開始用囉~~~  
先在server開啟websockify  
```bash
[root@120-114-142-26 ~]# websockify/websockify.py 5940 localhost:5900
websockify/websocket.py:30: UserWarning: no 'numpy' module, HyBi protocol will be slower
  warnings.warn("no 'numpy' module, HyBi protocol will be slower")
WebSocket server settings:
  - Listen on :5940
  - No SSL/TLS support (no cert file)
  - proxying from :5940 to localhost:5900
```

可以看到現在是收聽的狀態，可以注意到目前也沒有加密，如果要加密的的話可以在github上找方法  

開啟瀏覽器測試囉~~  
這邊要注意虛擬機的port和防火牆要開喔!!!!  
![](https://github.com/d93y70123123/classNote/blob/master/spice-html5.PNG)

