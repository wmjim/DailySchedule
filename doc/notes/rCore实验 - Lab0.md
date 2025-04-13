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