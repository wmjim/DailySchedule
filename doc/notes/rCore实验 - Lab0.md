rCore 实验 - Lab0

## Rust 开发环境配置

安装 Rust 相关软件包：

```bash
# 1. 安装 riscv64gc-unknown-none-elf 目标平台支持
rustup target add riscv64gc-unknown-none-elf

# 2. 安装 cargo-binutils 工具
cargo install cargo-binutils

# 3. 添加 llvm-tools-preview 组件
rustup component add llvm-tools-preview

# 4. 安装 rust-src 组件
rustup component add rust-src
```

- `riscv64gc-unknown-none-elf`：这使得 Rust 编译器能够针对 RISC-V 64 位架构生成目标代码，该目标平台通常用于裸机开发或嵌入式系统，没有操作系统或使用特定的嵌入式操作系统。
- `cargo-binutils`：该插件提供了与二进制工具集成的功能，例如可以将 Rust 代码编译生成的目标文件转换为其他格式，或者对目标文件进行反汇编、查看符号表等操作。
- `llvm-tools-preview`：该组件提供了 Rust 编译器对 LLVM 工具链的支持，包括 `rust-lld`、`rust-objcopy` 等工具。
- `rust-src`：该组件提供了 Rust 标准库的源代码，使得 Rust 编译器可以访问标准库的实现细节，例如可以使用 `std::mem::size_of` 函数获取类型的大小。

## Qemu 模拟器安装

```bash
# 1. 安装编译所需的依赖包
sudo apt install autoconf automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev \
              gawk build-essential bison flex texinfo gperf libtool patchutils bc \
              zlib1g-dev libexpat-dev pkg-config  libglib2.0-dev libpixman-1-dev git tmux python3

# 2. 下载源码包
wget https://download.qemu.org/qemu-7.0.0.tar.xz

# 3. 解压源码包
tar xvJf qemu-7.0.0.tar.xz

# 4. 编译安装并配置 RISC-v 支持
cd qemu-7.0.0
./configure --target-list=riscv64-softmmu,riscv64-linux-user
make -j$(nproc)

# 5. 将 Qemu 安装到 /usr/local/bin 目录下
sudo make install

# 或者将 Qemu 安装到 ~/.local/bin 目录下 （推荐，前提该目录以加入环境变量）
sudo make PREFIX=~/.local/bin install

# 6. 确认 Qemu 安装版本
$ qemu-system-riscv64 --version # 全新模拟器
QEMU emulator version 7.0.0
Copyright (c) 2003-2022 Fabrice Bellard and the QEMU Project developers
$ qemu-riscv64 --version    # 半系统模拟器：支持直接运行 risc-v 64 用户程序
qemu-riscv64 version 7.0.0
Copyright (c) 2003-2022 Fabrice Bellard and the QEMU Project developers
```