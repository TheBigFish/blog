<!-- en_title: ssh_protocol_simple analysis -->

# SSH 协议简单分析

简单分析 SSH 协议

<!-- more -->

## 服务器端配置

以ubuntu为例，配置文件位于 `/etc/ssh/sshd_config`
可以配置ssh服务端支持的认证算法，对应的公私钥文件位于 `etc/ssh/`

```bash
~$ tree /etc/ssh
/etc/ssh
├── moduli
├── ssh_config
├── sshd_config
├── ssh_host_dsa_key
├── ssh_host_dsa_key.pub
├── ssh_host_ecdsa_key
├── ssh_host_ecdsa_key.pub
├── ssh_host_ed25519_key
├── ssh_host_ed25519_key.pub
├── ssh_host_rsa_key
├── ssh_host_rsa_key.pub
└── ssh_import_id
```

## 验证服务端公钥

服务端公钥在 `dh gex reply（33)过程服务器发送host key (RSA公钥)` 时会将服务端对应的公钥下发，同时还有签名数据

![](https://github.com/TheBigFish/blog/raw/master/202002/ecdh_key_exchange_reply.png)

图中:
KEX host key 为类型 ssh-rsa 的公钥
kEX H signature 为服务端使用私钥进行签名后的数据

客户端收到该数据包后，到 known_hosts 文件寻找服务端 ip 对应的公钥，如果本地找不到对应的公钥数据，则提醒
客户端是不是信任远程主机

![](https://github.com/TheBigFish/blog/raw/master/202002/known_hosts_check.png)

确认后将该远程主机的公钥信息写入 known_hosts

![](https://github.com/TheBigFish/blog/raw/master/202002/known_hosts.png)

如果已经存在公钥数据但是跟当前服务端返回的公钥不一致，则客户端认为遭遇中间人攻击，断开连接

![](https://github.com/TheBigFish/blog/raw/master/202002/host_key_verification_failed.png)

