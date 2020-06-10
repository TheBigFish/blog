# grub加载

## 查看分区及挂载点

```bash
$ sudo fdisk -l

设备       启动     Start    末尾    扇区   Size Id 类型
/dev/sda1            2048   2000895   1998848   976M 83 Linux
/dev/sda2         2000896 234096639 232095744 110.7G 83 Linux
/dev/sda3       234098686 250068991  15970306   7.6G  5 扩展
/dev/sda5       234098688 249853951  15755264   7.5G 82 Linux 交换 / Solaris
/dev/sda6  *    249856000 250068991    212992   104M ef EFI (FAT-12/16/32)

~$ df
文件系统           1K-块     已用     可用 已用% 挂载点
udev             4029180        0  4029180    0% /dev
tmpfs             807200    17792   789408    3% /run
/dev/sda2      114094704 10741452 97534476   10% /
tmpfs            4035984        4  4035980    1% /dev/shm
tmpfs               5120        4     5116    1% /run/lock
tmpfs            4035984        0  4035984    0% /sys/fs/cgroup
/dev/sda1         967320   209452   691516   24% /boot
/dev/sda6         106270    56610    49660   54% /boot/efi
tmpfs             807200       40   807160    1% /run/user/1000
/dev/sdc1       20959232  3816944 17142288   19% /media/gwi/2834-6248
/dev/sdc2       39558124  1760420 37797704    5% /media/gwi/5D10CCB97866D434
```

```bash
root@gwi-FT2000:/home/gwi# tree /boot/efi -L 3
/boot/efi
├── boot
│   ├── grub
│   │   ├── arm64-efi
│   │   ├── fonts
│   │   ├── grub.cfg
│   │   ├── grubenv
│   │   └── locale
│   ├── Image
│   ├── initramfs.img
│   └── TRANS.TBL
├── EFI
│   ├── boot
│   │   ├── bootaa64.efi
│   │   └── grubaa64.efi
│   └── kylin
│       └── grubaa64.efi
└── TRANS.TBL
```
说明：

- efi 为单独分区 /dev/sda6,挂载与 /boot/efi，因为它的文件系统要被uefi识别，需要是 fat 文件系统
- uefi 对分区分别挂载
  ```
  /dev/sda1   FS0
  /dev/sda2   FS1
  /dev/sda3   FS2
  /dev/sda5   swap 分区，不识别
  /dev/sda6   FS3 efi 分区 
  ```
  从而在 uefi shell grub 文件位于 FS3
- grub efi 文件位于 */boot/efi/EFI/boot/bootaa64.efi*
- grub.cfg 文件位于 */boot/efi/boot/grub/grub.cfg*
  
## 手动选择 grub efi 文件
- uefi 界面按 F2 选择 boot device
- 选择 UEFI SHELL
- 输入 FS3:  
  `FS3:\>`
- 查看文件系统
```bash
  ls
> 输出  
  EFI/
  boot/
  TRANS.TBL
```
- 输入 cd EFI/BOOT
- 输入 bootaa64.efi 直接运行

对于uefi,他能够读取分区 FS3: ，并且能够直接运行对应的 grub efi 文件。该分区下面直接就有 EFI/ boot/ 文件夹。操作系统启动后将其挂载到 /boot/efi

