# rust 安装及配置

## 配置参数

```bash
vi ~/.bashrc

#add
export CARGO_HOME="~/.cargo/"
export RUSTBINPATH="~/.cargo/bin"
export RUST="~/.rustup/toolchains/stable-x86_64-unknown-linux-gnu"
export RUST_SRC_PATH="$RUST/lib/rustlib/src/rust/src"
export PATH=$PATH:$RUSTBINPATH
```

## 修改源

### .basrc

```bash
vi ~/.bashrc
#add
export RUSTUP_DIST_SERVER=https://mirrors.ustc.edu.cn/rust-static
export RUSTUP_UPDATE_ROOT=https://mirrors.ustc.edu.cn/rust-static/rustup
```

### config

```bash
vi ~/.cargo/config
#add
[source.crates-io]
replace-with = 'ustc'

[source.ustc]
registry = "git://mirrors.ustc.edu.cn/crates.io-index"
```

## ch02-00-guessing-game-tutorial.md 编译错误

```rust
// main.rst 文件顶部增加

extern crate rand;
```
