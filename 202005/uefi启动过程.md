
# uefi 启动过程

## uefi 找到 os loader

系统安装的最后阶段，grub 安装完，系统会调用 uefi 提供的接口，将grub 安装路径写入 uefi 变量。

uefi 通过该变量运行 grub uefi 文件。grub 再去引导内核。

## 打开调试

grub.cfg 增加一行  
`set debug=all`
