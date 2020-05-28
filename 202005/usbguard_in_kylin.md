# kylin usbguad 安装及配置分析

## 适配版本  

- [x] (FT2000-FTMBC V2.0) Kylin-4.0.2-desktop-sp3-19120521.Z1

## 软件包

- usbguard_0.7.2+ds-1kord0_arm64.deb
- usbguard-applet-qt_0.7.2+ds-1kord0_arm64.deb
- libusbguard0_0.7.2+ds-1kord0_arm64.deb

安装  
`sudo dpkg -i *.deb`  
缺乏依赖则使用  
`sudo apt install ` 安装缺失包

## 测试 usbguard

`sudo systemctl start usbguard`

`sudo usbguard list-devices`

```bash
gwi@gwi-FT2000:~/work$ sudo usbguard list-devices
1: allow id 1d6b:0002 serial "0000:04:00.0" name "xHCI Host Controller" hash "BXm6Ov4dzFbPriLpYudpl67dp9SgJJT5pwPN7LhqJjs=" parent-hash "s/FIoklYQ+H1MvoBvBtLiVG3V9UzeaxBo+b3EIDcZhk=" via-port "usb1" with-interface 09:00:00

.......

9: allow id 04d9:a0cd serial "" name "USB Keyboard" hash "9BPE2CEpJ3zS2TVXpYIg0v8ouwa24XJ91Lf5t3MK/Tc=" parent-hash "ryHCmG3nsLVuHD/YMplTUyPWzK2YMO368ASLReR84VQ=" via-port "5-1" with-interface { 03:01:01 03:00:00 03:00:00 }
12: allow id 09da:c10a serial "" name "USB Mouse" hash "ta0YgNdP2X6MQYUBvK02MPWaYLJmgTLP8Msle14ZI90=" parent-hash "0rFtia2crROTD3918CSuq/KDUocMqHMpalqjqQtbVgE=" via-port "7-3" with-interface 03:01:02
```

`sudo usbguard block-device 10`

测试成功

## 测试 d-bus

### 问题解决过程

检查发现 d-feet 已经安装  
搜索 usbguad 发现无法打开，对比正常版本，发现 `usbguard-dbus --system` 进程没有运行  
搜索发现安装目录下有 usbguard-dbus, 运行后，org.usbguard 正常打开。  
点击 listDevices，参数为 "allow", 获取设备失败。  
推测 d-bus 服务没有连接到 usbguad。
`systemctl stop usbguard` 之后， `systemctl start usbguard`, 再 `usbguard-dbus --system`,  
再使用 d-feet 获取 listDevice, 成功。  

原因分析：  
需要先启动 usbguard 服务，再启动 usbguard-dubs。

### 问题分析

ubuntu 18.04 apt 安装 usbguard，不存在上述问题。

1. 查看 服务

```bash
● usbguard.service - USBGuard daemon
   Loaded: loaded (/lib/systemd/system/usbguard.service; enabled; vendor preset: enabled)
   Active: active (running) since Wed 2020-05-27 18:40:00 CST; 9min ago
     Docs: man:usbguard-daemon(8)
  Process: 579 ExecStart=/usr/sbin/usbguard-daemon -f -s -c /etc/usbguard/usbguard-daemon.conf (code=exited, status=0/SUCCESS)
 Main PID: 654 (usbguard-daemon)
    Tasks: 3 (limit: 4664)
   CGroup: /system.slice/usbguard.service
           └─654 /usr/sbin/usbguard-daemon -f -s -c /etc/usbguard/usbguard-daemon.conf
5月 27 18:49:30 bigfish usbguard-daemon[654]: IPC connection denied: uid=1000 gid=1000 pid=3854
5月 27 18:49:31 bigfish usbguard-daemon[654]: IPC connection denied: uid=1000 gid=1000 pid=3854
5月 27 18:49:32 bigfish usbguard-daemon[654]: IPC connection denied: uid=1000 gid=1000 pid=3854
```

- enable, 开机启动
- usbguard-daemon 进程启动

分析 usbguard-daemon.conf, 无异常

2. 考虑是否有关联服务:

```bash
bigfish@bigfish:~$ sudo systemctl | grep usbguard
usbguard-dbus.service          loaded active running   USBGuard D-Bus Service
usbguard.service               loaded active running   USBGuard daemon 
```

ubuntu 上可以看到还有 usbguard-dbus 服务运行

```bash
bigfish@bigfish:~$ sudo systemctl status usbguard-dbus
● usbguard-dbus.service - USBGuard D-Bus Service
   Loaded: loaded (/lib/systemd/system/usbguard-dbus.service; enabled; vendor preset: enabled)
   Active: active (running) since Wed 2020-05-27 18:40:00 CST; 15min ago
     Docs: man:usbguard-dbus(8)
 Main PID: 704 (usbguard-dbus)
    Tasks: 4 (limit: 4664)
   CGroup: /system.slice/usbguard-dbus.service
           └─704 /usr/sbin/usbguard-dbus --system

5月 27 18:40:00 bigfish systemd[1]: Starting USBGuard D-Bus Service...
5月 27 18:40:00 bigfish systemd[1]: Started USBGuard D-Bus Service.
```

该服务开机运行，并启动了 `/usr/sbin/usbguard-dbus --system`

3. 分析 `usbguard-dbus.service`

```bash
bigfish@bigfish:~$ cat /lib/systemd/system/usbguard-dbus.service
[Unit]
Description=USBGuard D-Bus Service
Requires=usbguard.service
Documentation=man:usbguard-dbus(8)

[Service]
Type=dbus
BusName=org.usbguard
ExecStart=/usr/sbin/usbguard-dbus --system
Restart=on-failure

[Install]
WantedBy=multi-user.target
Alias=dbus-org.usbguard.service
```

服务依赖于 usbguard.service

## 问题解决

- 手动:  
    依次启动 usbguard-dbus.service 及 usbguard.service

- 开机启动：
    `sudo systemctl enable usbguard-dbus`
