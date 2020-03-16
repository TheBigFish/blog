<!-- en_title: usbguard compile and setting -->

# usbguanrd 编译和配置

## 编译

编译过程参考 [USBGuard on Ubuntu 16.04 #145](https://github.com/USBGuard/usbguard/issues/145)

```bash
git clone https://github.com/dkopecek/usbguard
sudo apt install libgcrypt11-dev libsodium-dev protobuf-compiler libprotobuf-dev libdbus-1-dev libdbus-glib-1-dev libpolkit-gobject-1-dev
cd usbguard/
./autogen.sh
./configure --prefix=/usr --sysconfdir=/etc --with-bundled-catch --with-bundled-pegtl --with-gui-qt=qt4 --enable-systemd
make
sudo make install
sudo ldconfig

sudo usbguard generate-policy > rules.conf
sudo install -m 0600 -o root -g root rules.conf /etc/usbguard/rules.conf

sudo systemctl restart usbguard

usbguard-applet-qt
```

中间遇到缺失包，直接安装

./configure 失败

![](https://github.com/TheBigFish/blog/raw/master/202003/modify_configure.png)
修改已经生成的 configure 文件
但是此方法编译成功后，service 无法启动

## 启动服务

- `sudo usbguard-daemon` 可以直接启动daemon服务。  
但是更加一般的做法是systemctl 启动服务。  

- `sudo systemctl start usbguard`:  
启动服务

- `sudo usbguard list-devices`:  

使用 usbguard 命令查看各种设备状态、策略

```shell
bigfish@bigfish:~$ sudo usbguard list-devices
10: allow id 1d6b:0002 serial "0000:00:0c.0" name "xHCI Host Controller" hash "ac+zTKYGulwXT3MzeXvz1GlYqGZHzg3xFUL9B7Sozwc=" parent-hash "y7wWUCqR/BAT5XHMsWDjNljffbA0AKeOMCxdoYG/q8E=" via-port "usb1" with-interface 09:00:00
11: allow id 1d6b:0003 serial "0000:00:0c.0" name "xHCI Host Controller" hash "9yYcgnuhCh7BD1Hm8GxdmIMISuf+DAW+iszxp3kSxFY=" parent-hash "y7wWUCqR/BAT5XHMsWDjNljffbA0AKeOMCxdoYG/q8E=" via-port "usb2" with-interface 09:00:00
12: allow id 80ee:0021 serial "" name "USB Tablet" hash "8S88DbsXkyb93aEG099kxcbjrHSGfpZEJ8W0048wl1A=" parent-hash "ac+zTKYGulwXT3MzeXvz1GlYqGZHzg3xFUL9B7Sozwc=" via-port "1-1" with-interface 03:00:00
```

- `sudo status usbguard.service`：  

查看服务状态

```shell
bigfish@bigfish:~$ sudo systemctl status usbguard
[sudo] password for bigfish:
● usbguard.service - USBGuard daemon
   Loaded: loaded (/lib/systemd/system/usbguard.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2020-03-16 18:40:08 CST; 2min 19s ago
     Docs: man:usbguard-daemon(8)
  Process: 699 ExecStart=/usr/sbin/usbguard-daemon -f -s -c /etc/usbguard/usbguard-daemon.conf (code=exited, status=0/SUCCESS)
 Main PID: 742 (usbguard-daemon)
    Tasks: 3 (limit: 4664)
   CGroup: /system.slice/usbguard.service
           └─742 /usr/sbin/usbguard-daemon -f -s -c /etc/usbguard/usbguard-daemon.conf

3月 16 18:42:18 bigfish usbguard-daemon[742]: IPC connection denied: uid=1000 gid=1000 pid=3848
```

## 配置

- usbguard-daemon.conf

配置守护服务

`/etc/usbguard/usbguard-daemon.conf`

修改后 restart 服务启动

- rules.conf

配置usb规则文件

`/etc/usbguard/rules.conf`

## D-Bus

D-Bus 的接口与官网描述不一致，生效的设置为：

```python

signal_object = self.bus.get_object('org.usbguard', '/org/usbguard/Devices')
devices_object = self.bus.get_object('org.usbguard', '/org/usbguard')
policy_object = self.bus.get_object('org.usbguard', '/org/usbguard/Devices')

self.signal_interface = dbus.Interface(signal_object, dbus_interface='org.usbguard.Devices')
self.devices_interface = dbus.Interface(devices_object, dbus_interface='org.usbguard.Devices')
self.policy_interface = dbus.Interface(policy_object, dbus_interface='org.usbguard.Policy')

self.signal_interface.connect_to_signal('DevicePresenceChanged', self.on_device_presence_changed)
```

## D-Bus 工具

### d-feet

### dbus-monitor

- 监听usb设备信号
`dbus-monitor --system`

```
bigfish@bigfish:~$ sudo dbus-monitor --system | grep DevicePresenceChanged
signal time=1584356432.006349 sender=:1.5 -> destination=(null destination) serial=3 path=/org/usbguard/Devices; interface=org.usbguard.Devices; member=DevicePresenceChanged
signal time=1584356438.308211 sender=:1.5 -> destination=(null destination) serial=5 path=/org/usbguard/Devices; interface=org.usbguard.Devices; member=DevicePresenceChanged
```

能够监听到u盘的插拔事件
