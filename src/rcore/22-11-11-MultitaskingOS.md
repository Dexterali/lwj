# 分时多任务系统--Multitasking OS

## 1.多道程序操作系统

用户程序的链接脚本

`build.py`

```python
# 构建应用程序的脚本
# 主要通过修改 user/linker.ld 中的 BASE_ADDRESS 完成

import os

base_address = 0x80400000
step = 0x20000  # 每个应用的最大大小
linker = 'src/linker.ld'

app_id = 0
apps = os.listdir('src/bin')
apps.sort()
for app in apps:
    app = app[:app.find('.')]
    lines = []
    lines_before = []
    with open(linker, 'r') as f:
        for line in f.readlines():
            lines_before.append(line)
            line = line.replace(hex(base_address), hex(base_address+step*app_id))
            lines.append(line)
    with open(linker, 'w+') as f:
        f.writelines(lines)
    os.system('cargo build --bin %s --release' % app)
    print('[build.py] application %s start with address %s' %(app, hex(base_address+step*app_id)))
    with open(linker, 'w+') as f:
        f.writelines(lines_before)
    app_id = app_id + 1
```

# 

## 2.协作式多任务操作系统

当 Trap 控制流准备调用 `__switch` 函数使任务从运行状态进入暂停状态的时候，它的内核栈上的情况

<img title="" src="http://rcore-os.cn/rCore-Tutorial-Book-v3/_images/task-context.png" alt="" data-align="center">

`__switch`的流程

<img title="" src="http://rcore-os.cn/rCore-Tutorial-Book-v3/_images/switch.png" alt="" data-align="center">

Trap 控制流在调用 `__switch` 之前就需要明确知道即将切换到哪一条目前正处于暂停状态的 Trap 控制流，因此 `__switch` 有两个参数，第一个参数代表它自己，第二个参数则代表即将切换到的那条 Trap 控制流。这里我们用上面提到过的 `current_task_cx_ptr` 和 `next_task_cx_ptr` 作为代表。在上图中我们假设某次 `__switch` 调用要从 Trap 控制流 A 切换到 B，一共可以分为四个阶段，在每个阶段中我们都给出了 A 和 B 内核栈上的内容。

> - 阶段 [1]：在 Trap 控制流 A 调用 `__switch` 之前，A 的内核栈上只有 Trap 上下文和 Trap 处理函数的调用栈信息，而 B 是之前被切换出去的；
> 
> - 阶段 [2]：A 在 A 任务上下文空间在里面保存 CPU 当前的寄存器快照；
> 
> - 阶段 [3]：这一步极为关键，读取 `next_task_cx_ptr` 指向的 B 任务上下文，根据 B 任务上下文保存的内容来恢复 `ra` 寄存器、`s0~s11` 寄存器以及 `sp` 寄存器。只有这一步做完后， `__switch` 才能做到一个函数跨两条控制流执行，即 *通过换栈也就实现了控制流的切换* 。
> 
> - 阶段 [4]：上一步寄存器恢复完成后，可以看到通过恢复 `sp` 寄存器换到了任务 B 的内核栈上，进而实现了控制流的切换。这就是为什么 `__switch` 能做到一个函数跨两条控制流执行。此后，当 CPU 执行 `ret` 汇编伪指令完成 `__switch` 函数返回后，任务 B 可以从调用 `__switch` 的位置继续向下执行。

从结果来看，我们看到 A 控制流 和 B 控制流的状态发生了互换， A 在保存任务上下文之后进入暂停状态，而 B 则恢复了上下文并在 CPU 上继续执行。

多道程序的典型执行情况

<img title="" src="http://rcore-os.cn/rCore-Tutorial-Book-v3/_images/multiprogramming.png" alt="" data-align="center">

task 状态变化图

<img title="" src="http://rcore-os.cn/rCore-Tutorial-Book-v3/_images/fsm-coop.png" alt="" data-align="center">

## 3.分时多任务与抢占式调度

RISC-V 中断一览表

| Interrupt | Exception Code | Description                   |
|:---------:|:--------------:|:-----------------------------:|
| 1         | 1              | Supervisor software interrupt |
| 1         | 3              | Machine software interrupt    |
| 1         | 5              | Supervisor timer interrupt    |
| 1         | 7              | Machine timer interrupt       |
| 1         | 9              | Supervisor external interrupt |
| 1         | 11             | Machine external interrupt    |

RISC-V 的中断可以分成三类：

- **软件中断** (Software Interrupt)：由软件控制发出的中断

- **时钟中断** (Timer Interrupt)：由时钟电路发出的中断

- **外部中断** (External Interrupt)：由外设发出的中断

在判断中断是否会被屏蔽的时候，有以下规则：

- 如果中断的特权级低于 CPU 当前的特权级，则该中断会被屏蔽，不会被处理；

- 如果中断的特权级高于 CPU 当前的特权级或相同，则需要通过相应的 CSR 判断该中断是否会被屏蔽。
