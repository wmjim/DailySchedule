# rCore实验 - Lab1

位于 ch1 分支上项目结构如下所示：

```bash
./os/src
Rust        4 Files   119 Lines
Assembly    1 Files    11 Lines

├── bootloader(内核依赖的运行在 M 特权级的 SBI 实现，本项目中我们使用 RustSBI)
│   └── rustsbi-qemu.bin(可运行在 qemu 虚拟机上的预编译二进制版本)
├── LICENSE
├── os(我们的内核实现放在 os 目录下)
│   ├── Cargo.toml(内核实现的一些配置文件)
│   ├── Makefile
│   └── src(所有内核的源代码放在 os/src 目录下)
│       ├── console.rs(将打印字符的 SBI 接口进一步封装实现更加强大的格式化输出)
│       ├── entry.asm(设置内核执行环境的的一段汇编代码)
│       ├── lang_items.rs(需要我们提供给 Rust 编译器的一些语义项，目前包含内核 panic 时的处理逻辑)
│       ├── linker-qemu.ld(控制内核内存布局的链接脚本以使内核运行在 qemu 虚拟机上)
│       ├── main.rs(内核主函数)
│       └── sbi.rs(调用底层 SBI 实现提供的 SBI 接口)
├── README.md
└── rust-toolchain(控制整个项目的工具链版本)
```

## 应用程序执行环境与平台支持

### 应用程序执行环境

> **应用程序的执行环境**（Execution Environment）是指运行一个应用程序时所需的硬
> 件、软件和配置的整体集合。它为应用程序提供了一个运行的“上下文”，确保程序能
> 正确加载、执行并与外部资源交互。简单来说，执行环境是应用程序运行的“舞台”，
> 包括所有必要的支持系统和条件。

![应用程序执行环境栈](../../picture/202504080001.png)

- **白色块**：表示各级执行环境
- **黑色块**：表示相邻两层执行环境之间的接口

### 目标平台与目标三元组

编译器编译、链接得到可执行文件时需要程序要在哪个平台上运行。

```bash
$ rustc --version --verbose
rustc 1.86.0 (05f9846f8 2025-03-31)
binary: rustc
commit-hash: 05f9846f893b09a1be1fc8560e33fc3c815cfecb
commit-date: 2025-03-31
host: x86_64-unknown-linux-gnu
release: 1.86.0
LLVM version: 19.1.7
```
默认平台：`x86_64-unknown-linux-gnu`

- CPU 架构：`x86_64`
- CPU 厂商：`unknown`
- 操作系统：`linux`
- 运行时库：`gnu`

查询目前 Rust 编译器支持哪些基于 RISC-V 的平台：

```bash
$ rustc --print target-list | grep riscv
riscv32-wrs-vxworks
riscv32e-unknown-none-elf
riscv32em-unknown-none-elf
riscv32emc-unknown-none-elf
riscv32gc-unknown-linux-gnu
riscv32gc-unknown-linux-musl
riscv32i-unknown-none-elf
riscv32im-risc0-zkvm-elf
riscv32im-unknown-none-elf
riscv32ima-unknown-none-elf
riscv32imac-esp-espidf
riscv32imac-unknown-none-elf
riscv32imac-unknown-nuttx-elf
riscv32imac-unknown-xous-elf
riscv32imafc-esp-espidf
riscv32imafc-unknown-none-elf
riscv32imafc-unknown-nuttx-elf
riscv32imc-esp-espidf
riscv32imc-unknown-none-elf
riscv32imc-unknown-nuttx-elf
riscv64-linux-android
riscv64-wrs-vxworks
riscv64gc-unknown-freebsd
riscv64gc-unknown-fuchsia
riscv64gc-unknown-hermit
riscv64gc-unknown-linux-gnu
riscv64gc-unknown-linux-musl
riscv64gc-unknown-netbsd
riscv64gc-unknown-none-elf
riscv64gc-unknown-nuttx-elf
riscv64gc-unknown-openbsd
riscv64imac-unknown-none-elf
riscv64imac-unknown-nuttx-elf
```

选择 `riscv64gc-unknown-none-elf` 作为目标平台，其中：
- CPU 架构：`riscv64`
- CPU 厂商：`unknown`
- 操作系统：`none`
- 运行时库：`elf`，表示没有标准的运行时库，但可以生成 ELF 格式的可执行文件

## 附录

### RustSBI

SBI（RISC-V Supervisor Binary Interface），OpenSBI 是 RISC-V 官方使用 C 语言开发的 SBI 参考实现；RustSBI 是 Rust 语言实现的 SBI 。

RustSBI 的作用：
1. 系统初始化与引导
2. 提供 SBI 服务
3. 支持多种环境
