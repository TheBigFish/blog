<!-- en_title: ubuntu12.04_bochs2.3.5_platform -->

# ubuntu12.04 上搭建 bochs2.3.5 调试环境

在 ubuntu12.04 64 位环境下使用源码编译 bochs2.3.5 带 debug 带 gui 版本。

<!-- more -->

## 环境搭建

操作系统 ubuntu12.04

### 替换源

1、首先备份Ubuntu12.04源列表

`sudo cp /etc/apt/sources.list /etc/apt/sources.list.backup`

2、修改更新源

`sudo gedit /etc/apt/sources.list`

```shell
# 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ precise main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ precise main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ precise-updates main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ precise-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ precise-backports main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ precise-backports main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ precise-security main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ precise-security main restricted universe multiverse

# 预发布软件源，不建议启用
# deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ precise-proposed main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ precise-proposed main restricted universe multiverse
```

3、更新源

`sudo apt-get update`

## bochs2.3.5

### 编译

```bash
tar vxzf bochs-2.3.5
cd bochs-2.3.5
./configure --enable-debugger --enable-disasm LDFLAGS=-L/usr/lib/i386-linux-gnu`
make
sudo make install
```

编译过程出错处理:  

- error: C compiler cannot create executables

  ```shell
  sudo apt-get install --reinstall build-essential
  sudo apt-get install --reinstall gcc
  sudo dpkg-reconfigure build-essential
  sudo dpkg-reconfigure gcc
  ```

- ERROR: X windows gui was selected, but X windows libraries were not found.
  `apt-get install xorg-dev`  
  仍然报错  

> 只要编译的时候连接了 -lX11这个库就可以了，所以可以让configure阶段出错的地方不退出，并且在make的时候link X11这个库，编辑configure, 将退出的地方注释掉  

> echo ERROR: X windows gui was selected, but X windows libraries were not found.
      #exit 1

> configure命令后加 LDFLAGS=-L/usr/lib/i386-linux-gnu  

>该问题不能用--with-nogui解决，否则无法输出hello os，因为需要使用gui
`./configure --enable-debugger --enable-disasm --enable-x86-64 LDFLAGS=-L/usr/lib/i386-linux-gnu`

#### make

make之前需要修改一份文件bx_debug/symbol.cc
在97行之后加入代码如下:

```cpp
using namespace std;

#ifdef __GNUC__ //修改
using namespace __gnu_cxx; //修改
#endif //修改

struct symbol_entry_t
```

## nasm 安装

`sudo apt-get install nasm`


## 测试

### asm 源码

boot.asm
``` asm
	org	07c00h			; 告诉编译器程序加载到7c00处
	mov	ax, cs
	mov	ds, ax
	mov	es, ax
	call	DispStr			; 调用显示字符串例程
	jmp	$			; 无限循环
DispStr:
	mov	ax, BootMessage
	mov	bp, ax			; ES:BP = 串地址
	mov	cx, 16			; CX = 串长度
	mov	ax, 01301h		; AH = 13,  AL = 01h
	mov	bx, 000ch		; 页号为0(BH = 0) 黑底红字(BL = 0Ch,高亮)
	mov	dl, 0
	int	10h			; 10h 号中断
	ret
BootMessage:		db	"Hello, OS world!"
times 	510-($-$$)	db	0	; 填充剩下的空间，使生成的二进制代码恰好为512字节
dw 	0xaa55				; 结束标志
```

### asm 源码编译

`nasm boot.asm -o boot.bin`

### 制作镜像

1.生成镜像文件

`bximage`

第一步选 fd, 其余默认
生成软盘镜像 a.img

2.写入引导扇区

`dd if=boot.bin of=a.img bs=512 count=1 conv=notrunc`

### 生成 bochsrc

``` shell
#how much memory the emulated machine will have
megs: 32

#filename of ROM images
romimage: file=$BXSHARE/BIOS-bochs-latest
vgaromimage: file=$BXSHARE/VGABIOS-lgpl-latest

#what disk images will be used
floppya: 1_44=a.img, status=inserted

#choose the boot disk
boot: floppy

#where do we send log messages?
log: bochsout.txt

#disable the mouse
mouse: enabled=0

#enable key mapping, using US layout as default.
keyboard_mapping: enabled=1, map=$BXSHARE/keymaps/x11-pc-us.map

```

### 运行

`bochs -f bochsrc`

- 输入回车
- 调试界面输入 c 回车， 运行

![](https://github.com/TheBigFish/blog/raw/master/202001/ubuntu12.04_bochs2.3.5_platform_effect.png)
