# 批处理系统--Batch System

<img title="" src="http://rcore-os.cn/rCore-Tutorial-Book-v3/_images/deng-fish.png" alt="" data-align="center">

## 1.特权级机制

什么时候会发生用户态到内核态的切换？

> 处理器检测到用户态执行环境中执行了内核态特权级指令

软硬件协同设计

> 1. 执行环境调用指令`ecall`：具有用户态到内核态的执行环境切换能力的函数调用指令
> 
> 2. 执行环境返回指令`eret`：具有内核态到用户态的执行环境切换能力的函数返回指令

## 2.Risc-V特权级架构

| 级别  | 编码  | 名称                            |
|:---:|:---:|:-----------------------------:|
| 0   | 00  | 用户/应用模式 (U, User/Application) |
| 1   | 01  | 监督模式 (S, Supervisor)          |
| 2   | 10  | 虚拟监督模式 (H, Hypervisor)        |
| 3   | 11  | 机器模式 (M, Machine)             |

## 3.执行环境栈与特权级

<img title="" src="http://rcore-os.cn/rCore-Tutorial-Book-v3/_images/PrivilegeStack.png" alt="" data-align="center">

## 4.Risc-V定义的异常

| Interrupt | Exception Code | Description                    |
|:---------:|:--------------:|:------------------------------:|
| 0         | 0              | Instruction address misaligned |
| 0         | 1              | Instruction access fault       |
| 0         | 2              | Illegal instruction            |
| 0         | 3              | Breakpoint                     |
| 0         | 4              | Load address misaligned        |
| 0         | 5              | Load access fault              |
| 0         | 6              | Store/AMO address misaligned   |
| 0         | 7              | Store/AMO access fault         |
| 0         | 8              | Environment call from U-mode   |
| 0         | 9              | Environment call from S-mode   |
| 0         | 11             | Environment call from M-mode   |
| 0         | 12             | Instruction page fault         |
| 0         | 13             | Load page fault                |
| 0         | 15             | Store/AMO page fault           |

## 5.特权级切换

<img title="" src="http://rcore-os.cn/rCore-Tutorial-Book-v3/_images/EnvironmentCallFlow.png" alt="" data-align="center">

## 6.Risc-V S 模式特权指令

RISC-V S模式特权指令

| 指令              | 含义                                                                                                                                                                         |
|:---------------:|:--------------------------------------------------------------------------------------------------------------------------------------------------------------------------:|
| sret            | 从 S 模式返回 U 模式：在 U 模式下执行会产生非法指令异常                                                                                                                                           |
| wfi             | 处理器在空闲时进入低功耗状态等待中断：在 U 模式下执行会产生非法指令异常                                                                                                                                      |
| sfence.vma      | 刷新 TLB 缓存：在 U 模式下执行会产生非法指令异常                                                                                                                                               |
| 访问 S 模式 CSR 的指令 | 通过访问 [sepc/stvec/scause/sscartch/stval/sstatus/satp等CSR](http://rcore-os.cn/rCore-Tutorial-Book-v3/chapter2/4trap-handling.html#term-s-mod-csr) 来改变系统状态：在 U 模式下执行会产生非法指令异常 |

## 7.Risc-V 寄存器编号和别名以及作用

`risc-v`寄存器编号从`0~31`，表示为`x0~x31`。其中

+ `x10~x17`对应`a0~a7`

+ `x1`对应`ra`

`risc-v`调用规范约定`a0~a6`保存系统调用的参数，`a0`保存系统调用的返回值，而`a7`用来传递`syscall ID`

## 8.`link_app.S`应用程序链接脚本

```nasm
# os/src/link_app.S
    .align 3
    .section .data
    .global _num_app

_num_app:
    .quad 5
    .quad app_0_start
    .quad app_1_start
    .quad app_2_start
    .quad app_3_start
    .quad app_4_start
    .quad app_4_end

    .section .data
    .global app_0_start
    .global app_0_end

app_0_start:
    .incbin "../user/target/riscv64gc-unknown-none-elf/release/00hello_world.bin"
app_0_end:

    .section .data
    .global app_1_start
    .global app_1_end

app_1_start:
    .incbin "../user/target/riscv64gc-unknown-none-elf/release/01store_fault.bin"
app_1_end:

    .section .data
    .global app_2_start
    .global app_2_end

app_2_start:
    .incbin "../user/target/riscv64gc-unknown-none-elf/release/02power.bin"
app_2_end:

    .section .data
    .global app_3_start
    .global app_3_end

app_3_start:
    .incbin "../user/target/riscv64gc-unknown-none-elf/release/03priv_inst.bin"
app_3_end:

    .section .data
    .global app_4_start
    .global app_4_end

app_4_start:
    .incbin "../user/target/riscv64gc-unknown-none-elf/release/04priv_csr.bin"
app_4_end:
```

## 9. 批处理操作系统提供的`AEE`（应用程序执行环境），需要做到：

> - 当启动应用程序的时候，需要初始化应用程序的用户态上下文，并能切换到用户态执行应用程序；
> 
> - 当应用程序发起系统调用（即发出 `Trap`）之后，需要到批处理操作系统中进行处理；
> 
> - 当应用程序执行出错的时候，需要到批处理操作系统中杀死该应用并加载运行下一个应用；
> 
> - 当应用程序执行结束的时候，需要到批处理操作系统中加载运行下一个应用（实际上也是通过系统调用 `sys_exit` 来实现的）。

## 10.特权级切换相关的控制状态寄存器`CSR`

+ 进入 S 特权级 `Trap` 的相关 `CSR`

| CSR 名   | 该 CSR 与 Trap 相关的功能                        |
|:-------:|:-----------------------------------------:|
| sstatus | `SPP` 等字段给出 Trap 发生之前 CPU 处在哪个特权级（S/U）等信息 |
| sepc    | 当 Trap 是一个异常的时候，记录 Trap 发生之前执行的最后一条指令的地址  |
| scause  | 描述 Trap 的原因                               |
| stval   | 给出 Trap 附加信息                              |
| stvec   | 控制 Trap 处理代码的入口地址                         |

## 11.特权级切换的硬件控制机制

当 CPU 执行完一条指令（如 `ecall` ）并准备从用户特权级 陷入（ `Trap` ）到 S 特权级的时候，硬件会自动完成如下这些事情：

> - `sstatus` 的 `SPP` 字段会被修改为 CPU 当前的特权级（U/S）。
> 
> - `sepc` 会被修改为 Trap 处理完成后默认会执行的下一条指令的地址。
> 
> - `scause/stval` 分别会被修改成这次 Trap 的原因以及相关的附加信息。
> 
> - CPU 会跳转到 `stvec` 所设置的 Trap 处理入口地址，并将当前特权级设置为 S ，然后从Trap 处理入口地址处开始执行。

> 注解：**stvec 相关细节**
> 
> 在 RV64 中， `stvec` 是一个 64 位的 CSR，在中断使能的情况下，保存了中断处理的入口地址。它有两个字段：
> 
> - MODE 位于 [1:0]，长度为 2 bits；
> 
> - BASE 位于 [63:2]，长度为 62 bits。
> 
> 当 MODE 字段为 0 的时候， `stvec` 被设置为 Direct 模式，此时进入 S 模式的 Trap 无论原因如何，处理 Trap 的入口地址都是 `BASE<<2` ， CPU 会跳转到这个地方进行异常处理。本书中我们只会将 `stvec` 设置为 Direct 模式。而 `stvec` 还可以被设置为 Vectored 模式，有兴趣的同学可以自行参考 RISC-V 指令集特权级规范。

而当 CPU 完成 Trap 处理准备返回的时候，需要通过一条 S 特权级的特权指令 `sret` 来完成，这一条指令具体完成以下功能：

- CPU 会将当前的特权级按照 `sstatus` 的 `SPP` 字段设置为 U 或者 S ；

- CPU 会跳转到 `sepc` 寄存器指向的那条指令，然后继续执行。

## 12.Trap 管理

- 应用程序通过 `ecall` 进入到内核状态时，操作系统保存被打断的应用程序的 Trap 上下文；

- 操作系统根据Trap相关的CSR寄存器内容，完成系统调用服务的分发与处理；

- 操作系统完成系统调用服务后，需要恢复被打断的应用程序的Trap 上下文，并通 `sret` 让应用程序继续执行。

Trap 上下文的保存和恢复

`trap.S`：

```nasm
.altmacro
.macro SAVE_GP n
    sd x\n, \n*8(sp)
.endm
.macro LOAD_GP n
    ld x\n, \n*8(sp)
.endm
    .section .text
    .globl __alltraps
    .globl __restore
    .align 2 # 将 _alltraps 的地址 4 字节对齐
__alltraps: # trap 处理程序的入口点
    csrrw sp, sscratch, sp
    # now sp->kernel stack, sscratch->user stack
    # allocate a TrapContext on kernel stack
    addi sp, sp, -34*8
    # save general-purpose registers
    sd x1, 1*8(sp)
    # skip sp(x2), we will save it later
    sd x3, 3*8(sp)
    # skip tp(x4), application does not use it
    # save x5~x31
    .set n, 5
    .rept 27
        SAVE_GP %n
        .set n, n+1
    .endr
    # we can use t0/t1/t2 freely, because they were saved on kernel stack
    csrr t0, sstatus
    csrr t1, sepc
    sd t0, 32*8(sp)
    sd t1, 33*8(sp)
    # read user stack from sscratch and save it on the kernel stack
    csrr t2, sscratch
    sd t2, 2*8(sp)
    # set input argument of trap_handler(cx: &mut TrapContext)
    mv a0, sp
    call trap_handler   # rust 实现 trap 的分发和处理 

__restore:  # 恢复 trap 之前的寄存器状态
    # case1: start running app by __restore
    # case2: back to U after handling trap
    mv sp, a0
    # now sp->kernel stack(after allocated), sscratch->user stack
    # restore sstatus/sepc
    ld t0, 32*8(sp)
    ld t1, 33*8(sp)
    ld t2, 2*8(sp)
    csrw sstatus, t0
    csrw sepc, t1
    csrw sscratch, t2
    # restore general-purpuse registers except sp/tp
    ld x1, 1*8(sp)
    ld x3, 3*8(sp)
    .set n, 5
    .rept 27
        LOAD_GP %n
        .set n, n+1
    .endr
    # release TrapContext on kernel stack
    addi sp, sp, 34*8
    # now sp->kernel stack, sscratch->user stack
    csrrw sp, sscratch, sp
    sret
```

> - 第 7 行我们使用 `.align` 将 `__alltraps` 的地址 4 字节对齐，这是 RISC-V 特权级规范的要求；
> 
> - `csrrw` 原型是 `csrrw rd, csr, rs` 可以将 CSR 当前的值读到通用寄存器 `rd` 中，然后将通用寄存器 `rs` 的值写入该 CSR 。因此这里起到的是交换 sscratch 和 sp 的效果。在这一行之前 sp 指向用户栈， sscratch 指向内核栈（原因稍后说明），现在 sp 指向内核栈， sscratch 指向用户栈。
> 
> - 第 12 行，我们准备在内核栈上保存 Trap 上下文，于是预先分配 `34 * 8 `字节的栈帧，这里改动的是 sp ，说明确实是在内核栈上。
> 
> - 第 13~24 行，保存 Trap 上下文的通用寄存器 x0~x31，跳过 x0 和 tp(x4)，原因之前已经说明。我们在这里也不保存 sp(x2)，因为我们要基于它来找到每个寄存器应该被保存到的正确的位置。实际上，在栈帧分配之后，我们可用于保存 Trap 上下文的地址区间为 `[sp, sp + 8 * 34]`，按照 `TrapContext` 结构体的内存布局，基于内核栈的位置（sp所指地址）来从低地址到高地址分别按顺序放置 x0~x31这些通用寄存器，最后是 sstatus 和 sepc 。因此通用寄存器 xn 应该被保存在地址区间 `[sp + 8n, sp + 8(n + 1)]` 。为了简化代码，x5~x31 这 27 个通用寄存器我们通过类似循环的 `.rept` 每次使用 `SAVE_GP` 宏来保存，其实质是相同的。注意我们需要在 `trap.S` 开头加上 `.altmacro` 才能正常使用 `.rept` 命令。
> 
> - 第 25~28 行，我们将 CSR sstatus 和 sepc 的值分别读到寄存器 t0 和 t1 中然后保存到内核栈对应的位置上。指令 `csrr rs, csr` 的功能就是将 CSR 的值读到寄存器 `rd` 中。这里我们不用担心 t0 和 t1 被覆盖，因为它们刚刚已经被保存了。
> 
> - 第 30~31 行专门处理 sp 的问题。首先将 sscratch 的值读到寄存器 t2 并保存到内核栈上，注意： sscratch 的值是进入 Trap 之前的 sp 的值，指向用户栈。而现在的 sp 则指向内核栈。
> 
> - 第 33 行令 `a0 <- sp`，让寄存器 a0 指向内核栈的栈指针也就是我们刚刚保存的 Trap 上下文的地址，这是由于我们接下来要调用 `trap_handler` 进行 Trap 处理，它的第一个参数 `cx` 由调用规范要从 a0 中获取。而 Trap 处理函数 `trap_handler` 需要 Trap 上下文的原因在于：它需要知道其中某些寄存器的值，比如在系统调用的时候应用程序传过来的 syscall ID 和对应参数。我们不能直接使用这些寄存器现在的值，因为它们可能已经被修改了，因此要去内核栈上找已经被保存下来的值

```nasm
__restore:  # 恢复 trap 之前的寄存器状态
    # case1: start running app by __restore
    # case2: back to U after handling trap
    mv sp, a0
    # now sp->kernel stack(after allocated), sscratch->user stack
    # restore sstatus/sepc
    ld t0, 32*8(sp)
    ld t1, 33*8(sp)
    ld t2, 2*8(sp)
    csrw sstatus, t0
    csrw sepc, t1
    csrw sscratch, t2
    # restore general-purpuse registers except sp/tp
    ld x1, 1*8(sp)
    ld x3, 3*8(sp)
    .set n, 5
    .rept 27
        LOAD_GP %n
        .set n, n+1
    .endr
    # release TrapContext on kernel stack
    addi sp, sp, 34*8
    # now sp->kernel stack, sscratch->user stack
    csrrw sp, sscratch, sp
    sret
```

> - 第 4 行比较奇怪我们暂且不管，假设它从未发生，那么 sp 仍然指向内核栈的栈顶。
> 
> - 第 7~20 行负责从内核栈顶的 Trap 上下文恢复通用寄存器和 CSR 。注意我们要先恢复 CSR 再恢复通用寄存器，这样我们使用的三个临时寄存器才能被正确恢复。
> 
> - 在第 22 行之前，sp 指向保存了 Trap 上下文之后的内核栈栈顶， sscratch 指向用户栈栈顶。我们在第 28 行在内核栈上回收 Trap 上下文所占用的内存，回归进入 Trap 之前的内核栈栈顶。第 30 行，再次交换 sscratch 和 sp，现在 sp 重新指向用户栈栈顶，sscratch 也依然保存进入 Trap 之前的状态并指向内核栈栈顶。
> 
> - 在应用程序控制流状态被还原之后，第 31 行我们使用 `sret` 指令回到 U 特权级继续运行应用程序控制流。

注解：**sscratch CSR 的用途**

> 在特权级切换的时候，我们需要将 Trap 上下文保存在内核栈上，因此需要一个寄存器暂存内核栈地址，并以它作为基地址指针来依次保存 Trap 上下文的内容。但是所有的通用寄存器都不能够用作基地址指针，因为它们都需要被保存，如果覆盖掉它们，就会影响后续应用控制流的执行。
> 
> 事实上我们缺少了一个重要的中转寄存器，而 `sscratch` CSR 正是为此而生。从上面的汇编代码中可以看出，在保存 Trap 上下文的时候，它起到了两个作用：首先是保存了内核栈的地址，其次它可作为一个中转站让 `sp` （目前指向的用户栈的地址）的值可以暂时保存在 `sscratch` 。这样仅需一条 `csrrw  sp, sscratch, sp` 指令（交换对 `sp` 和 `sscratch` 两个寄存器内容）就完成了从用户栈到内核栈的切换，这是一种极其精巧的实现。

trap 分发和处理

```rust
pub fn trap_handler(cx: &mut TrapContext) -> &mut TrapContext {
    let scause = scause::read(); // get trap cause
    let stval = stval::read(); // get extra value
    match scause.cause() {
        Trap::Exception(Exception::UserEnvCall) => {
            cx.sepc += 4;
            // 从 a0 读取整个 TrapContext
            // 这里就体现了操作系统和用户程序约定的系统调用规范
            cx.x[10] = syscall(cx.x[17], [cx.x[10], cx.x[11], cx.x[12]]) as usize;
        }
        Trap::Exception(Exception::StoreFault) | Trap::Exception(Exception::StorePageFault) => {
            println!("[kernel] PageFault in application, kernel killed it.");
            run_next_app();
        }
        Trap::Exception(Exception::IllegalInstruction) => {
            println!("[kernel] IllegalInstruction in application, kernel killed it.");
            run_next_app();
        }
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

> + 第 4 行声明返回值为 `&mut TrapContext` 并在第 25 行实际将传入的Trap 上下文 `cx` 原样返回，因此在 `__restore` 的时候 `a0` 寄存器在调用 `trap_handler` 前后并没有发生变化，仍然指向分配 Trap 上下文之后的内核栈栈顶，和此时 `sp` 的值相同，这里的- 并不会有问题；
> - 第 7 行根据 `scause` 寄存器所保存的 Trap 的原因进行分发处理。这里我们无需手动操作这些 CSR ，而是使用 Rust 的 riscv 库来更加方便的做这些事情。
> 
> - 第 8~11 行，发现触发 Trap 的原因是来自 U 特权级的 Environment Call，也就是系统调用。这里我们首先修改保存在内核栈上的 Trap 上下文里面 sepc，让其增加 4。这是因为我们知道这是一个由 `ecall` 指令触发的系统调用，在进入 Trap 的时候，硬件会将 sepc 设置为这条 `ecall` 指令所在的地址（因为它是进入 Trap 之前最后一条执行的指令）。而在 Trap 返回之后，我们希望应用程序控制流从 `ecall` 的下一条指令开始执行。因此我们只需修改 Trap 上下文里面的 sepc，让它增加 `ecall` 指令的码长，也即 4 字节。这样在 `__restore` 的时候 sepc 在恢复之后就会指向 `ecall` 的下一条指令，并在 `sret` 之后从那里开始执行。
>   
>   用来保存系统调用返回值的 a0 寄存器也会同样发生变化。我们从 Trap 上下文取出作为 syscall ID 的 a7 和系统调用的三个参数 a0~a2 传给 `syscall` 函数并获取返回值。 `syscall` 函数是在 `syscall` 子模块中实现的。 这段代码是处理正常系统调用的控制逻辑。
> 
> - 第 12~20 行，分别处理应用程序出现访存错误和非法指令错误的情形。此时需要打印错误信息并调用 `run_next_app` 直接切换并运行下一个应用程序。
> 
> - 第 21 行开始，当遇到目前还不支持的 Trap 类型的时候，“邓式鱼” 批处理操作系统整个 panic 报错退出。

## 13.执行应用程序

当批处理操作系统初始化完成，或者是某个应用程序运行结束或出错的时候，我们要调用 `run_next_app` 函数切换到下一个应用程序。此时 CPU 运行在 S 特权级，而它希望能够切换到 U 特权级。在 RISC-V 架构中，唯一一种能够使得 CPU 特权级下降的方法就是执行 Trap 返回的特权指令，如 `sret` 、`mret` 等。事实上，在从操作系统内核返回到运行应用程序之前，要完成如下这些工作：

- 构造应用程序开始执行所需的 Trap 上下文；

- 通过 `__restore` 函数，从刚构造的 Trap 上下文中，恢复应用程序执行的部分寄存器；

- 设置 `sepc` CSR的内容为应用程序入口点 `0x80400000`；

- 切换 `scratch` 和 `sp` 寄存器，设置 `sp` 指向应用程序用户栈；

- 执行 `sret` 从 S 特权级切换到 U 特权级。
