# codeblock 链接 openssl 库文件

## 安装 codeblock
    版本为： codeblocks-17.12mingw-setup.exe
    gcc: 5.1.0

## 安装 MSYS-1.0.10.exe
    将 `CodeBlocks\MinGW` 下文件拷贝至 `msys\1.0\mingw`
    gcc -v 输出版本号即成功

## 安装 perl 5
    strawberry-perl-5.30.0.1-64bit.msi

## 编译 openssl
    - 版本 openssl-1.0.2j.tar.gz
    - 编译
        - ./config
        - make
        - make test
        - make install
    编译后文件位于 `msys\1.0\local\ssl`

## 配置codeblock
    - project -> build options -> linker settings 增加：
        - libcrypto.a
        - libssl.a
        - libgdi32.a
    - project -> build options -> search directories 增加：
        - msys\1.0\local\ssl\include

完成 