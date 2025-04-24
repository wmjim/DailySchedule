# rCore 实验 - Lab2

位于 ch2 分支上项目结构如下所示：

```bash
./os/src
Rust        13 Files   372 Lines
Assembly     2 Files    58 Lines

├── bootloader
│   └── rustsbi-qemu.bin
├── LICENSE
├── os
│   ├── build.rs(新增：生成 link_app.S 将应用作为一个数据段链接到内核)
│   ├── Cargo.toml
│   ├── Makefile(修改：构建内核之前先构建应用)
│   └── src
│       ├── batch.rs(新增：实现了一个简单的批处理系统)
│       ├── console.rs(将打印字符的 SBI 接口进一步封装实现更加强大的格式化输出)
│       ├── entry.asm(设置内核执行环境的一段汇编代码)
│       ├── lang_items.rs(需要我们提供给 Rust 编译器的一些语义项，包含内核 panic 时的处理逻辑)
│       ├── link_app.S(构建产物，由 os/build.rs 输出)
│       ├── linker-qemu.ld(控制内核内存布局的链接脚本以使得内核运行在 qemu 虚拟机上)
│       ├── main.rs(修改：主函数中需要初始化 Trap 处理并加载和执行应用)
│       ├── sbi.rs(调用底层 SBI 实现提供的 SBI 接口)
│       ├── sync(新增：同步子模块 sync ，目前唯一功能是提供 UPSafeCell)
│       │   ├── mod.rs
│       │   └── up.rs(包含 UPSafeCell，它可以帮助我们以更 Rust 的方式使用全局变量)
│       ├── syscall(新增：系统调用子模块 syscall)
│       │   ├── fs.rs(包含文件 I/O 相关的 syscall)
│       │   ├── mod.rs(提供 syscall 方法根据 syscall ID 进行分发处理)
│       │   └── process.rs(包含任务处理相关的 syscall)
│       └── trap(新增：Trap 相关子模块 trap)
│           ├── context.rs(包含 Trap 上下文 TrapContext)
│           ├── mod.rs(包含 Trap 处理入口 trap_handler)
│           └── trap.S(包含 Trap 上下文保存与恢复的汇编代码)
├── README.md
├── rust-toolchain(控制整个项目的工具链版本)
└── user(新增：应用测例保存在 user 目录下)
   ├── Cargo.toml
   ├── Makefile
   └── src
      ├── bin(基于用户库 user_lib 开发的应用，每个应用放在一个源文件中)
      │   ├── 00hello_world.rs
      │   ├── 01store_fault.rs
      │   ├── 02power.rs
      │   ├── 03priv_inst.rs
      │   └── 04priv_csr.rs
      ├── console.rs
      ├── lang_items.rs
      ├── lib.rs(用户库 user_lib)
      ├── linker.ld(应用的链接脚本)
      └── syscall.rs(包含 syscall 方法生成实际用于系统调用的汇编指令，
                     各个具体的 syscall 都是通过 syscall 来实现的)
```

## 特权级机制

### RISC-V 的指令

| 指令         | 含义                                                   |
| ------------ | ------------------------------------------------------ |
| `call`       | **伪指令**，用于函数调用，子程序跳转                   |
| `ecall`      | **基础指令**，用于触发从用户模式到更高特权级的软中断   |
| `ret`        | **伪指令**，用于从函数调用处返回                       |
| `sret`       | **特权级指令**，用于从 S-Mode 的异常和中断处理程序返回 |
| `eret`       | **最高特权级指令**，用于从 M-Mode 的异常和中断程序返回 |
| `wfi`        | **特权级指令**，处理器在空闲时进入低功耗状态等待中断   |
| `sfence.vma` | **特权级指令**，刷新 TLB 缓存                          |

### 特权级软硬件协同设计

低特权级软件只能做高特权级允许它做的，且超出低特权级软件能力的功能必须寻求高特权级软件的帮助，高特权级软件（操作系统）称为低特权级软件（一般软件）的软件执行环境的重要组成部分。

为实现这样的特权级机制，需要软硬件协同设计。

### RISC-V 特权级架构

| 级别 | 编码 | 名称                                |
| ---- | ---- | ----------------------------------- |
| 0    | 00   | 用户/应用模式 (U, User/Application) |
| 1    | 01   | 监督模式 (S, Supervisor)            |
| 2    | 10   | 虚拟监督模式 (H, Hypervisor)        |
| 3    | 11   | 机器模式 (M, Machine)               |

本系列实验设计实现 RISC-V 的 M/S/U 三种特权级：
- **U 模式**：应用程序和用户态支持库运行在 U-Mode 的最低特权级。
- **S 模式**：操作系统内核运行在 S-Mode 特权级，形成支持应用程序和用户态支持库的执行环境。
- **M 模式**：RustSBI 运行在更底层的 M-Mode 特权级，是操作系统内核的执行环境。

![alt text](../../picture/202504130001.png)

## 实现应用程序

### 应用程序设计

应用程序的目录文件结构：

```bash
tree user
├── Cargo.lock
├── Cargo.toml
├── Makefile
└── src 
    ├── bin # bin 下的 *.rs 文件是各个应用程序
    │   ├── 00hello_world.rs
    │   ├── 01store_fault.rs
    │   ├── 02power.rs
    │   ├── 03priv_inst.rs
    │   └── 04priv_csr.rs
    ├── console.rs 		# I/O 函数用户库
    ├── lang_items.rs 	# 
    ├── lib.rs 			# 入口函数、初始化函数
    ├── linker.ld 		# 应用程序的内存说明
    └── syscall.rs 		# 系统调用函数用户库
```

- `user/src/bin/*.rs`：各个应用程序
- `user/src/*.rs`：用户库
- `user/src/linker.ld`：应用程序的内存说明

### 项目结构

`user/bin/*.rs` 里包含多个应用程序，批处理操作系统会按照文件名开头的数字编号依次顺序加载并运行。

1、在代码中尝试引入外部用户库：

```rust
#[macro_use]
extern crate user_lib;
```

2、在 `lib.rs` 中定义用户库的入口点 `_start`

```rust
#![feature(linkage)] // 支持 linkage 属性，linkage 属于 nightly 

...

#[unsafe(no_mangle)]
#[unsafe(link_section = ".text.entry")]
pub extern "C" fn _start() -> ! {
    clear_bss();  // 手动清零需要初始化的 .bss 段
    exit(main()); // 首先在 bin 目录下寻找 main, 找不到则使用 lib.rs 里的 main
    panic!("unreachable after sys_exit!");
}

...

// 弱链接, 找不到 main 时会使用这个
#[linkage = "weak"]
#[unsafe(no_mangle)]
fn main() -> i32 {
    panic!("Cannot find main!");
}
```

- `#![feature(linkage)]`：支持各种链接操作
- `#[unsafe(link_section = ".text.entry")]` 将 `_start` 这段代码编译后的汇编代码放在 `.text.entry` 的代码段中。

- 弱链接的作用：如果在 `bin` 目录下找不到 `main`，允许编译通过，运行时会 `panic`。

### 内存布局

使用链接脚本 `user/src/linker.ld`，进行内存布局如下：

- 将程序的起始物理地址调整为 `0x8040, 0000`，应用程序加载到该物理地址上运行；
- 将 `_start` 所在的 `.text.entry` 放在整个程序的开头——批处理系统加载之后跳转到 `0x8040, 0000` 

### 系统调用

```rust
// user/src/syscall.rs
use core::arch::asm;
fn syscall(id: usize, args: [usize; 3]) -> isize {
    let mut ret: isize;
    unsafe {
        asm!(
            "ecall", // 触发系统调用异常
            inlateout("x10") args[0] => ret, // 输入参数1/返回值寄存器
            in("x11") args[1],  // 输入参数2寄存器  
            in("x12") args[2],  // 输入参数3寄存器
            in("x17") id        // 系统调用号寄存器
        );
    }
    ret
}
```
工作流程：
1. 通过寄存器传递参数（`x10`-`x12`传系统调用参数，`x17`传系统调用号）
2. 执行 `ecall` 指令后，CPU 会跳转到内核预设的异常处理程序
3. 内核完成后，返回值通过 `x10`（`a0`）寄存器返回用户态

## 实现特权级的切换

### 特权级切换相关的控制状态寄存器

| CSR 名    | 该 CSR 与 Trap 相关的功能                                    |
| --------- | ------------------------------------------------------------ |
| `sstatus` | `SPP` 等字段给出 Trap 发生之前 CPU 处于哪个特权级（S/U）等信息 |
| `sepc`    | 当 Trap 是一个异常的时候，记录 Trap 发生之前执行的最后一条指令的地址 |
| `scause`  | 描述 Trap 的原因                                             |
| `stval`   | 给出 Trap 的附加信息                                         |
| `stvec`   | 控制 Trap 处理代码的入口地址                                 |

### 特权级切换的硬件控制机制

**CPU 从用户态特权级 Trap 到 S 特权级**，硬件会自动完成如下事情：

1. `sstatus` 的 `SPP` 字段会被修改为 CPU 当前的特权级；
2. `sepc` 会被修改为 Trap 处理完成后默认会执行的下一条指令的地址；
3. `scause` 被修改为本次 Trap 的原因；
4. `stval` 被修改为本次 Trap 的相关附加信息；
5. CPU 会跳转到 `stvec` 所设置的 Trap 处理地址入口，并将当前特权级设置为 S，然后从 Trap 处理入口地址开始执行；

CPU 完成 Trap 处理准备返回时，通过一条 S 特权级的特权指令 `sret` 来完成，该指令具有以下功能：

- CPU 会将 `sstatus` 的 `SPP` 字段设置为 U 或者 S;
- CPU 会跳转到 `sepc` 寄存器指向的那条指令，然后继续执行；

### Trap 管理

#### Trap 上下文的保存与恢复

在操作系统初始化的时候，修改 `stvec` 寄存器以指向 Trap 处理入口点：

```rust
// os/src/trap/mod.rs

/// 定义 trap 处理的汇编代码
global_asm!(include_str!("trap.S"));

/// 初始化 trap 处理
pub fn init() {
    extern "C" {
        // 声明外部汇编函数 __alltraps，这是所有 trap 的统一入口点
        fn __alltraps();
    }
    unsafe {
        // 将 __alltraps 地址写入 stvec 寄存器，并设置 trap 模式为 Direct
        stvec::write(__alltraps as usize, TrapMode::Direct);
    }
}
```

- `os/src/trap/trap.S`：实现 Trap 上下文保存 / 恢复的汇编代码，通过 `global_asm!` 将 `trap.S` 这段汇编代码插入进来。

**保存 Trap 上下文的 `__alltraps` 的实现**：

```assembly
# 将指定的通用寄存器保存在内核栈上
.macro SAVE_GP n
    sd x\n, \n*8(sp)
.endm

.align 2 # 确保 __alltraps 入口地址是 4 字节对齐的（RISC-V 要求 trap 处理程序地址至少 4 字节对齐）
__alltraps:
    # 交换 sp 和 sscratch 寄存器值
    # 执行后：sp->内核栈, sscratch->用户栈
    csrrw sp, sscratch, sp
    # 在内核栈上分配 TrapContext 结构空间（34个8字节单元）
    addi sp, sp, -34*8
    
    # 保存通用寄存器：
    sd x1, 1*8(sp)
    # 跳过sp（x2），稍后会保存它
    sd x3, 3*8(sp)
    # 跳过tp（x4），应用程序不使用它
    # 使用宏循环保存 x5-x31 寄存器
    .set n, 5
    .rept 27
        SAVE_GP %n
        .set n, n+1
    .endr

    # 可以自由使用t0/t1/t2，因为它们保存在内核堆栈上
    # 保存控制状态寄存器：
    csrr t0, sstatus
    csrr t1, sepc
    sd t0, 32*8(sp)
    sd t1, 33*8(sp)
    # 保存用户栈指针（原存在 sscratch 中）
    csrr t2, sscratch
    sd t2, 2*8(sp)  # 保存到 TrapContext 的 sp 位置

    # 准备调用 Rust 陷阱处理函数
    mv a0, sp # 设置 trap_handler 的输入参数（cx：&mut TrapContext）
    call trap_handler # 调用 Rust 实现的处理函数
    
```

- `.align` 将 `__alltraps` 的地址 4 字节对齐，这是 RISC-V 特权级规范的要求
- `csrrw sp, sscratch, sp`：这里起到交换 `sscratch` 和 `sp` 的效果。交换之前 `sp` → 用户栈，`sscratch` → 内核栈；交换之后 `sp` → 内核栈，`sscratch` → 用户栈。
- `addi sp, sp, -34*8`：预分配 34 * 8 字节（刚好是一个 `TrapContext` 结构体在内存中占用大小）的栈帧
- 保存通用寄存器 `x0` ~ `x31`，跳过 `x0` 和 `tp(x4)`。通用寄存器被保存在地址区间 `[sp + 8n, sp + s(n+1))` 。
- 第28~第31行分别读取 `sstatus` 和 `sepc` 的值先读取到 `t0` 和 `t1` 再保存到内核栈对应的位置上。
- 第33~第34行专门处理用户栈 `sp` 的值。先将 `sscratch` 的值读取到 `t2` 再保存到内核栈。
- `mv a0, sp` 让寄存器 `a0` 指向内核栈的栈指针，接下来调用 `trap_handler` 进行 Trap 处理，它的第一个参数 `cx` 由于调用规范将从 `a0` 中获取。
- [以上寄存器内存分配的相对位置](###寄存器在内核栈保存的相对位置)。

### Trap 分发与处理

下面介绍使用 Rust 实现的 `trap_handler`，在该函数中完成 Trap 的分发与处理：

```rust
/// trap handler
#[no_mangle]
pub fn trap_handler(cx: &mut TrapContext) -> &mut TrapContext {
    // 读取 trap 原因和附加信息
    let scause = scause::read();    // 获取trap原因(scause寄存器)
    let stval = stval::read();          // 获取附加信息(stval寄存器)
    // trace!("into {:?}", scause.cause());
    match scause.cause() {
        // 处理系统调用
        Trap::Exception(Exception::UserEnvCall) => {
            // jump to next instruction anyway
            cx.sepc += 4;
            // get system call return value
            cx.x[10] = syscall(cx.x[17], [cx.x[10], cx.x[11], cx.x[12]]) as usize;
        }
        // 处理页错误/存储错误
        Trap::Exception(Exception::StoreFault) | Trap::Exception(Exception::StorePageFault) => {
            println!("[kernel] PageFault in application, bad addr = {:#x}, bad instruction = {:#x}, kernel killed it.", stval, cx.sepc);
            exit_current_and_run_next();
        }
        // 处理非法指令异常
        Trap::Exception(Exception::IllegalInstruction) => {
            println!("[kernel] IllegalInstruction in application, kernel killed it.");
            exit_current_and_run_next();
        }
        // 处理定时器中断
        Trap::Interrupt(Interrupt::SupervisorTimer) => {
            set_next_trigger();
            suspend_current_and_run_next();
        }
        // 其他未处理的陷阱类型
        _ => {
            panic!(
                "Unsupported trap {:?}, stval = {:#x}!",
                scause.cause(),
                stval
            );
        }
    }
    cx
}
```

- 传入的 `cx` T`a0` 仍指向分配 Trap 上下文之后的内核栈顶。
- 根据 `scause` 寄存器所保存的 Trap 原因进行分发处理。



## 附录

### 寄存器在内核栈保存的相对位置

```bash
栈偏移(8字节单位) | 保存内容
----------------|-------------------
0               | (未使用)
1               | x1 (ra) 返回地址
2               | x2 (sp) 用户栈指针
3               | x3 (gp) 全局指针
4               | (跳过 x4/tp)
5               | x5 (t0)
6               | x6 (t1)
7               | x7 (t2)
8               | x8 (s0/fp)
9               | x9 (s1)
10              | x10 (a0)
11              | x11 (a1)
12              | x12 (a2)
13              | x13 (a3)
14              | x14 (a4)
15              | x15 (a5)
16              | x16 (a6)
17              | x17 (a7)
18              | x18 (s2)
19              | x19 (s3)
20              | x20 (s4)
21              | x21 (s5)
22              | x22 (s6)
23              | x23 (s7)
24              | x24 (s8)
25              | x25 (s9)
26              | x26 (s10)
27              | x27 (s11)
28              | x28 (t3)
29              | x29 (t4)
30              | x30 (t5)
31              | x31 (t6)
32              | sstatus 状态寄存器
33              | sepc 异常PC
```

1. 总共分配了 34*8 字节空间(对应 `TrapContext` 结构)
2. `x0(zero)` 不需要保存(始终为 0 )
3. `x2(sp)` 的特殊处理：通过 `sscratch` 交换后单独保存
4. `x4(tp)` 被明确跳过(应用不使用)
5. 控制寄存器保存在最后两个位置(`sstatus` 和 `sepc`)
6. 这种布局与 Rust 中的 `TrapContext` 结构定义完全对应
