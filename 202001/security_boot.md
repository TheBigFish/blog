# uefi security boot 实现细节

## 密钥体系

![](https://github.com/TheBigFish/blog/raw/master/assets/security_boot_01.png)

### PKpub
> Platform Key

- 确定平台所有者和平台固件之间的信任关系
- BIOS厂商/OEM创建这个KEY
- PK是自签名
- PKpriv的所有者必须保证私钥的安全 

### KEKpub
> Key Exchange Key 

- 确定平台固件和操作系统之间的信任关系
- KEY 用来对db/dbx进行签名
- 操作系统厂商提供
- OEM也可提供自己的KEY对shell app进行签名

### DB
> 合规数据库

- 用来存放运行被执行的代码的签名，或者存储证书
- OEM可以把IHV(独立硬件厂商）的签名放在DB里面，用来对第三方OpROMs（UEFI CA)进行校验
- MS CA / UEFI CA 由 MSFT提供.

### DBX
> 禁止数据库

- 用来存储禁止执行代码的签名
- 或者被禁止的公司的证书


### KEY的生成过程

![](https://github.com/TheBigFish/blog/raw/master/assets/security_boot_02.png)

db 及 dbx 存放的是记录以及由某个 KEK 对该记录进行签名的数据，记录可以是：
1. efi 文件的hash 值
2. efi 文件的签名数据
3. 对 efi 文件进行签名的证书

PK 由平台所有者持有，比如联想
KEK 由PK签名

db/dbx 数据进行更新时，bios需要进行授权验证，验证该更新数据是否是由某个 KEK 进行签名，若验证通过，则将该条记录及其对应的签名数据存入 db/dbx。

MSCApri 以及 UEFICApri 可以对厂商提供的驱动进行签名。

#### 驱动更新流程

比如某个显卡厂商Card

* 使用自己的私钥 (Cardpri) 对自己的驱动进行签名生成 CardSign，(其证书为 CardCert)。

* 找 UEFICApri 对自己的数据（CardSign、CardCert）进行签名，生成 (（CardSign，UEFICApri_CardSign_Sign ）（CardCert，UEFICApri_CardCert_Sign ）） 。

* 调用bios接口进行更新，更新数据为 (（CardSign，UEFICApri_CardSign_Sign ）（CardCert，UEFICApri_CardCert_Sign ））

* bios 中有 UEFICApub 对应的 KEY (UEFIKEKpub)

* bios 遍历 KEK，使用KEK对 (（CardSign，UEFICApri_CardSign_Sign ）（CardCert，UEFICApri_CardCert_Sign ））进行验签

* 验签通过，将条目 （CardSign，UEFICApri_CardSign_Sign）、（CardCert，UEFICApri_CardCert_Sign）存入 db

#### efi 文件无签名数据的情况

按照上述校验流程，显卡厂商也可以不使用自己的私钥对驱动签名，而只是生成一个哈希值。
再使用 UEFICApri 对哈希值进行签名生成  (CardHash, UEFICApri_CardHash_Sign)。

要将该哈希及对应签名信息更新到 db 时，仍然要使用 KEK 验证 UEFICApri_CardHash_Sign。

使用hash值的efi文件在启动时，只需要验证该 efi 文件的hash 是否匹配 db/dbx 中的某个hash。

#### 驱动校验流程

* bios 验证显卡驱动
* 显卡驱动中由对应的签名的证书（CardCert）信息。
* 遍历 db 找到 CardCert
* 使用 CardCert 对 db 中所有类型为签名值的条目进行验证
* 如果某条验证通过，则校验通过

#### dbx

dbx内容与db中数据格式一致，匹配dbx中数据则认为验证失败


#### MSCApri 与 UEFICApri
市面上已经有大量使用 MSCApri 以及 UEFICApri 签名的 efi。

要加入该体系，只需要，使用 PKpri 对 MSCApub 及 UEFICApub 进行签名，生成对应的两个 KEK:  MSKEKpub  UEFIKEKpub。

将该两个KEK 加入 bios ,即可以支持已签名的 efi。

#### OS Loader
> 比如grub

OS Loader 使用 MSCApri 签名，对应的 MSCApub 存储于 db，同时 MSKEKpub 也内置于 bios。


### db/dbx 防篡改

db/dbx 每条记录都是 数据以及使用 KEK 对该数据的签名。
bios 启动时，逐条遍历记录并验证签名。


## 概括
通过使用分级密钥体系
* 平台所有者只持有 PKpri
* 各级板卡厂商及操作系统厂商可以使用各自的公私钥体系对数据进行签名
* 平台所有者通过签名各级厂商的公钥生成 KEK, 将平台所有者认为可信的厂商加入信任