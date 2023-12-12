# `MIT0.S081`

## Lec01 Introduction and Examples

## Lec03 OS Organization and System Calls

#### 操作系统隔离性

##### 应用程序之间的隔离

使不同的应用程序之间不会相互影响

##### 应用程序和操作系统的隔离

操作系统不会因为应用程序而崩溃

##### 应用程序和硬件资源的隔离

#### 操作系统防御性

防止应用程序直接控制内核

#### 硬件对于强隔离的支持

##### `user/kernel mode`

###### `user mode`

当运行在user mode时，CPU只能运行普通权限的指令

###### `kernel mode`

当运行在kernel mode时，CPU可以运行特定权限的指令（主要是一些直接操纵硬件的指令和设置保护的指令）

##### 虚拟内存

处理器包含了page table，而page table将虚拟内存地址与物理内存地址做了对应。

每一个进程都会有自己独立的page table，这样的话，每一个进程只能访问出现在自己page table中的物理内存。操作系统会设置page table，使得每一个进程都有不重合的物理内存，这样一个进程就不能访问其他进程的物理内存，因为其他进程的物理内存都不在它的page table中。一个进程甚至都不能随意编造一个内存地址，然后通过这个内存地址来访问其他进程的物理内存。

#### `user / kernel mode` 切换

用户空间运行的程序运行在`user mode`，内核空间的程序运行在`kernel mode`

`ecall`

#### 宏内核 vs 微内核

##### 宏内核

让整个操作系统代码都运行在 `kernel mode`

###### 注意

内核中有大量的代码，出现bug的可能性变大

将所有子模块紧密集成在一起，提供了不错的性能

##### 微内核

减少内核中的代码

###### 注意

`user / kernel mode` 之间反复切换带来的性能损失

微内核将各个组成部分隔离开，使得 `page cache` 的共享更难实现，更难在微内核中得到更高的性能

#### 编译运行kernel

`xv6` 包括三个部分

* `kernel: kernel mode` 的程序 
* `user: user mode` 的程序 
* `mkfs` 空文件镜像，一个空的文件系统。

`Makefile` 会为所有内核文件做相同的操作，之后，系统加载器（`Loader`）会收集所有的`.o` 文件，将它们链接在一起，并生成内核文件。这里生成的内核文件就是我们将会在 `QEMU` 中运行的文件

#### `QEMU`

但是在内部，在 `QEMU` 的主循环中，只在做一件事情：

- 读取 `4` 字节或者 `8` 字节的 `RISC-V` 指令。
- 解析`RISC-V`指令，并找出对应的操作码（`op code`）。我们之前在看 `kernel.asm` 的时候，看过一些操作码的二进制版本。通过解析，或许可以知道这是一个 `ADD` 指令，或者是一个 `SUB` 指令。
- 之后，在软件中执行相应的指令。

```cpp
for (; ;) {
	read instruction
	decode instruction
	execute instruction
}
```

#### `XV6` 启动过程

`XV6` 从 `entry.s` 开始启动，这个时候没有内存分页，没有隔离性，并且运行在 `M-mode`。

然后尽可能快的跳转到 `kernel mode` 

然后执行 `main` 进行内核的初始化

```
kinit(); 设置好页表分配器（page allocator）
kvminit(); 设置好虚拟内存，这是下节课的内容
kvminithart(); 打开页表，也是下节课的内容
processinit(); 设置好初始进程或者说设置好进程表单
trapinit/trapinithart(); 设置好user/kernel mode转换代码
plicinit/plicinithart(); 设置好中断控制器PLIC（Platform Level Interrupt Controller），我们后面在介绍中断的时候会详细的介绍这部分，这是我们用来与磁盘和console交互方式
binit(); 分配buffer cache
iinit(); 初始化inode缓存
fileinit(); 初始化文件系统
virtio_disk_init(); 初始化磁盘
userinit(); 最后当所有的设置都完成了，操作系统也运行起来了，会通过userinit运行第一个进程，这里有点意思，接下来我们看一下userinit
```

`userinit` 初始化了第一个用户进程（`initcode`），将 `init` 地址加载到 `a0`，将 `argv` 的地址加载到 `a1`，将 `exec` 系统调用号加载到 `a7`，最后调用 `ECALL` 然后将控制权交给操作系统并转入 `syscall` 函数中

```
...             内核空间
userinit        内核空间 -> 用户空间
执行上面三条指令
第四条指令       用户空间 -> 内核空间
```

在 `syscall`函数中告诉内核某个用户程序执行了 `ECALL` 指令，并且想要调用 `exec` 系统调用，然后执行系统调用`p->trapframe->a0 = syscall[num]()` 得到对应的 `exec` 入口函数，然后跳转到 `sys_exec` 函数中，`sys-exec` 首先从用户空间读取参数，拷贝到内核空间，在这里就是将 `init` 程序传入，所以 `initcode` 完成了 `exec` 调用 `init` 程序，`init` 为用户空间设置好 `console`，并调用 `fork`，在 `fork` 的子进程中执行 `shell`，最后将 `shell` 运行起来了

## Lec04 Page tables

#### 页表

实现在一个物理内存上，创建不同的地址空间

通过处理器和内存管理单元实现

#### `MMU` 内存管理单元

##### `virtual address`

| 25   | 27    | 12     |
| ---- | ----- | ------ |
| EXT  | index | offset |

##### `page table` 

| 44   | 10    |
| ---- | ----- |
| PPN  | Flags |

##### `physical address`

| 44   | 12     |
| ---- | ------ |
| PPN  | Offset |

一个 `page table` 最大为 $2^{27}$ ，物理地址最大为 $2^{56}$

<img src="C:\Users\tyl\Desktop\2022下半年\计网\img\assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKKjB2an4WcuUmOlE__%2F-MKPwJezGQDkWaLDRuDs%2Fimage.png" alt="img" style="zoom: 33%;" />

`SATP` 寄存器会指向最高一级的 `page directory` 的物理内存地址，之后我们用虚拟内存中 `index` 的高 `9bit` 用来索引最高一级的 `page directory`，这样我们就能得到一个`PPN`，也就是物理 `page` 号。这个 `PPN` 指向了中间级的 `page directory`。

当我们在使用中间级的 `page directory` 时，我们通过虚拟内存地址中的 `L1` 部分完成索引。接下来会走到最低级的 `page directory`，我们通过虚拟内存地址中的 `L0` 部分完成索引。在最低级的 `page directory` 中，我们可以得到对应于虚拟内存地址的物理内存地址。

#### 页表缓存（`TLB`）

最近使用过的虚拟地址的翻译结果有缓存

#### `Kernel Page Table`

主板上电后，第一、件事情是运行存储在 `boot ROM` 的代码，`boot` 完成后跳转到地址 `0x80000000`，

<img src="C:\Users\tyl\Desktop\2022下半年\计网\img\assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MK_UbCc81Y4Idzn55t8%2F-MKaY9xY8MaH5XTiwuBm%2Fimage.png" alt="img" style="zoom:50%;" />

#### `kvminit`（物理地址）

`kvminit` 是 `main` 中调用的第一个函数，这个函数设置好了 `kernel` 的地址空间，通过 `kvmmap` 将物理地址映射到相同的虚拟地址

然后通过 `vmprint` 函数打印完整的 `kernel page directory`

#### `kvminithart` 

```c
w_satp(MAKE_SATP(kernel_pagetable));
```

这条指令之后，开始地址翻译，这条指令之前是物理内存中，这条地址之后就是虚拟内存中了

#### `walk`

`walk` 函数模拟了 `MMU` 完成了地址翻译的过程

## Lec06 Isolation & system call entry/exit

#### `Trap` 机制

用户空间和内核空间的切换

1. `Trap` 最开始， `CPU` 所有状态都设置成运行用户代码

2. 保存 32 位用户寄存器
3. 保存程序寄存器
4. 将 `mode` 切换为 `supervisor mode`
5. 将 `SATP` 指向 `kernel page table` 
6. 将堆栈寄存器指向内核的一个位置
7. 跳转到内核的 `C` 代码

##### 注意

不能让用户代码介入到这里的切换，`xv6` 的 `trap` 机制不会查看这些寄存器，只会保存

`supervisor mode` 只能让你做到读写控制寄存器和使用 `PTE_U` 标指位为 0 的 `PTE`

`supervisor mode` 并不能读写任意物理地址

#### `Trap` 代码执行流程

![image-20231121145632161](C:\Users\tyl\Desktop\2022下半年\计网\img\image-20231121145632161.png)

#### `uservec`

`xv6` 为每个 `user page table` 映射了一个 `trapframe page` 来保存用户寄存器

从 `ECALL` 进入 `uervec` 时，会将寄存器 `SSCRATCH` 和 `a0` 进行互换，`SCRATCH` 里面的就是 `trapfram page` 的起始地址

#### `usertrap`

`usertrap` 某种程度上存储并恢复硬件状态，但是它也需要检查触发 `trap` 的原因，以确定相应的处理方式

## Lec08 Page faults

#### `Page Fault Basics`

通过 `page fault`，内核可以更新 `page table`

出现 `page fault` 时，我们想要知道

* 出错的虚拟地址，触发 `page fault` 的源，当出现page fault的时候，XV6内核会打印出错的虚拟地址，并且这个地址会被保存在STVAL寄存器中。
* 出错的原因，`load, store, jump`，如果是因为 `page fault` 触发的 `trap` 机制并且进入到内核空间，`STVAL` 寄存器和`SCAUSE` 寄存器都会有相应的值。
* 触发 `page fault` 指令的地址，`trap` 处理代码时，将这个地址存放在 `SEPC` 中

#### `Lazy page allocation`

##### `sbrk`

用户应用程序通过 `sbrk` 扩大自己的 `heap` 

`xv6` 中，`sbrk` 一旦调用立即分配应用程序，应用程序往往会申请多于自己需要的内存

##### `lazy allocation`

`sbrk` 系统调基本上不做任何事情，唯一需要做的事情就是提升`p->sz`，将 `p->sz`增加 `n`，其中 `n` 是需要新分配的内存 `page` 数量。但是内核在这个时间点并不会分配任何物理内存。之后在某个时间点，应用程序使用到了新申请的那部分内存，这时会触发 `page fault`，因为我们还没有将新的内存映射到 `page table`。

当我们看到一个 `page fault` 时，如果这个虚拟地址处于 `p->sz`，和 `stack` 之间，那么我们就知道这是一个 `heap` 的地址，但是没有分配物理地址，这个时候我们只需要在 `page fault handler` 中通过 `kalloc` 分配一个内存 `page`，然后初始化 `page` 为 0，然后映射到 `user page table` 中，最后重新执行指令

####  `Zero Fill On Demand`

如果有许多个 `page`，他们的内容全为 `0`，那么我们只需要分配一个全 `0` 的 `page`，然后将所有 `page` 指向它

这个 `page` 只有 `PTE_R` 权限，当我们需要修改指向它的一个 `page` 的时候，需要为这个 `page` 重新分配内存 `page`，然后将内容指定为 `0`，并更新两个 `page` 间的 `mapping` 关系，重新设置 `PTE`，最后重新执行指令

##### 优点

节省内存

程序启动更快，可以得到更好的交互体验

#### `Copy On Write Fork`

`shell` 处理指令时，会通过 `fork` 创建一个子进程，子进程会完整拷贝父进程的地址空间，然后调用 `exec` 运行其他程序，`exec` 会丢弃这个地址空间，然后变成一个包含其他程序的地址空间。

当我们创建子进程时，与其创建，分配并拷贝内容到新的物理内存，其实我们可以直接共享父进程的物理内存 `page`。所以这里，我们可以设置子进程的 `PTE` 指向父进程对应的物理内存 `page`。为了保证进程间的隔离性，我们要将父进程和子进程的 `PTE` 标志位都设置成只读的。

需要修改时，我们会得到一个 `page fault`，得到 `page fault` 之后，我们分配一个新的物理 `page`，拷贝相应的物理 `page`，然后将新 `page` 的 `PTE` 设置成可读写的，这时父进程的物理 `page` 只对父进程可见，所以也应变成可读可写的。

##### 注意

我们需要修改 `PTE` 标志位最后两位 `RSW`，用于区分是否发生的是 `copy on write`。

我们需要对于每一个物理内存 `page` 的引用进行计数，当我们释放虚拟 `page` 时，我们将物理内存 `page` 的引用数减 `1`，如果引用数等于 `0`，那么我们就能释放物理内存 `page`。

#### `Demand Paging`

在需求分页中，操作系统将程序执行所需的页面（页框）从磁盘加载到物理内存中，以满足程序的实际需求。这种方式延迟了页面加载的时间，只有在程序真正需要时才进行加载，从而减少了开销并提高了内存利用率。

对于 `demand paging` 来说，假设内存已经耗尽了或者说 `OOM` 了，这个时候如果得到了一个 `page fault`，需要从文件系统拷贝中拷贝一些内容到内存中，但这时你又没有任何可用的物理内存 `page`，这个时候我们可以使用 `LRU` 撤回 `page`，使用 `LRU` 时，我们通常选择 `non-dirty-page` 进行替换，因为在替换 `dirty page` 时，我们需要先将其写回磁盘中，再将其和我们当前需要加载的 `page` 进行替换，同时，我们会依靠 `Access bit` 来实现 `LRU` 策略。

学生提问：那是不是要定时的将 `Access bit` 恢复成 `0`？

Frans教授：是的，这是一个典型操作系统的行为。操作系统会扫描整个内存，这里有一些著名的算法例如 `clock algorithm`，就是一种实现方式。

另一个学生提问：为什么需要恢复这个 `bit`？

Frans教授：如果你想知道 `page` 最近是否被使用过，你需要定时比如每 `100` 毫秒或者每秒清除 `Access bit`，如果在下一个`100` 毫秒这个 `page` 被访问过，那你就知道这个 `page` 在上一个 `100` 毫秒中被使用了。而 `Access bit` 为 `0` 的 `page` 在上 `100` 毫秒未被使用。这样你就可以统计每个内存 `page` 使用的频度，这是一个成熟的 `LRU` 实现的基础。（注，可以通过 `Access bit` 来决定内存 `page` 在 `LRU` 中的排名）

#### `Memory Mapped Files`

将完整或者部分文件加载到内存中，这样就可以通过内存地址相关的 `load` 或者 `store` 指令来操纵文件。为了支持这个功能，一个现代的操作系统会提供一个叫做 `mmap` 的系统调用。这个系统调用会接收一个虚拟内存地址（`VA`），长度（`len`），`protection`，一些标志位，一个打开文件的文件描述符，和偏移量（`offset`）。

从文件描述符对应的文件的偏移量的位置开始，映射长度为 `len` 的内容到虚拟内存地址 `VA`，同时我们需要加上一些保护，比如只读或者读写

## Lec09 Interrupts

#### 真实操作系统内存使用情况

大部分操作系统运行时几乎没有任何空闲的内存。

#### 在XV6中设置中断

`SIE` 寄存器，一个 `bit(E)` 用来针对 UART 外部设备的中断，`bit(S)` 用来针对软件中断，`bit(T) ` 用来针对定时器中断

`SSTATUS` 寄存器，有一个 `bit` 用来打开或关闭中断

##### 步骤

使用 `consoleinit()` 设置第一个外设 `console`，在 `consoleinit()` 中先初始化了锁。然后调用 `uartinit()` 将 UART 芯片设置好使其可以被使用，具体流程是是先关闭中断，之后设置波特率，设置字符长度为 `8bit`，重置 FIFO，最后再重新打开中断。

此时还没有对 PLIC 编程，中断不能被感知，所以我们接下来需要调用 `plicinit()` 函数，这是 0 号 CPU 运行的，之后的 CPU 的核调用 `plicinithart()` 表明对哪些中断感兴趣

然后调用 `scheduler()` ，`scheduler()` 先执行 `intr_on()` 来让 CPU 可以接收中断，然后实际运行进程 

## Lec10 Multiprocessors and locking

#### 锁如何避免 `race condition`？

锁是一个对象，里面包含 acquier 和 release API，在这两个函数中间的代码要么一起进行，要么都不进行，这样就能避免 race condition

####  什么时候使用锁？

如果两个进程访问了一个共享的数据结构，并且其中一个进程会更新共享的数据结构，那么就需要对于这个共享的数据结构加锁。

#### 锁的特性和死锁

锁可以避免丢失更新。

锁可以打包多个操作，使它们具有原子性。

锁可以维护共享数据结构的不变性。

## Lec11 Thread switching 

#### 线程（Thread）概述

线程就是单个串行执行代码的单元，它只占用一个CPU并且以普通的方式一个接一个的执行指令。

线程状态包括三个部分

* 程序计数器，当前线程执行指令的位置
* 保存变量的寄存器
* 程序的 `Stack`，记录了函数调用记录，反应了当前线程的执行点

多线程并行策略：

* 一个 `CPU` ，多个线程来回切换
* 多个 `CPU` 同时处理

内核线程共享内存，用户进程不共享内存

#### XV6线程调度

过程：

用户进程`->` 中断器强制拿走 CPU 控制权 `->` 让给线程调度器 `->` 转交给其他线程

在 xv6 中：

定时器中断会强制的将CPU控制权从用户进程给到内核，这里是 `pre-emptive scheduling`，之后内核会代表用户进程（注，实际是内核中用户进程对应的内核线程会代表用户进程出让CPU），使用 `voluntary scheduling`。

三类线程

* RUNNING，线程当前正在某个 CPU 上运行

* RUNABLE，线程还没有在某个 CPU 上运行，但是一旦有空闲的 CPU 就可以运行

* SLEEPING，这个状态意味着线程在等待一些 I/O 事件，它只会在 I/O 事件发生了之后运行

#### XV6线程切换（一）

**当XV6从 CC 程序的内核线程切换到 LS 程序的内核线程时：**

1. XV6会首先会将 CC 程序的内核线程的内核寄存器保存在一个 `context` 对象中。
2. 类似的，因为要切换到 LS 程序的内核线程，那么 LS 程序现在的状态必然是 RUNABLE，表明 LS 程序之前运行了一半。这同时也意味着 LS程序的用户空间状态已经保存在了对应的 `trapframe` 中，更重要的是，LS 程序的内核线程对应的内核寄存器也已经保存在对应的 `context` 对象中。所以接下来，XV6 会恢复 LS 程序的内核线程的 `context` 对象，也就是恢复内核线程的寄存器。
3. 之后 LS 会继续在它的内核线程栈上，完成它的中断处理程序（注，假设之前 LS 程序也是通过定时器中断触发的 `pre-emptive scheduling` 进入的内核）。
4. 然后通过恢复 LS 程序的 `trapframe` 中的用户进程状态，返回到用户空间的 LS 程序中。
5. 最后恢复执行 LS。

**这里核心点在于，在 XV6 中，任何时候都需要经历：**

1. 从一个用户进程切换到另一个用户进程，都需要从第一个用户进程接入到内核中，保存用户进程的状态并运行第一个用户进程的内核线程。
2. 再从第一个用户进程的内核线程切换到第二个用户进程的内核线程。
3. 之后，第二个用户进程的内核线程暂停自己，并恢复第二个用户进程的用户寄存器。
4. 最后返回到第二个用户进程继续执行。

####  XV6线程切换（二）

用户进程 中断寄存器产生中断，保存 `trapframe`

`usertrap` 执行相应的中断处理程序

调用 `switch` 函数

`switch` 函数将当前用户进程对应的内核线程的寄存器至 `context` 中，然后切换到这个 CPU 的调度器线程，恢复之前保存的寄存器和 `stack pointer`，然后调用 `schedulder` 函数

`schedulder` 函数将当前进程设置为 RUNABLE，然后找到下一个 RUNABLE 进程，再次使用 `switch` 函数 

`switch` 函数将当前调度器线程的寄存器保存到 `context` 中，然后恢复找到的进程的寄存器和 `stack pointer`

因为之前也使用了`switch` 函数，所以我们现在要完成 `switch`，然后处理由当前进程切换时产生的中断处理程序

然后返回用户空间，恢复 `trapframe` 中的用户进程的寄存器和 `stack pointer`

最后进程顺利运行

## Lec13 Sleep & Wake up

## Lec14 File systems

#### `File system` 实现概述

首先出现在接口中的路径名是可读的名字，而不是一串数字，它是由用户选择的字符串。

`write` 系统调用并没有使用 `offset` 作为参数，所以写入到文件的哪个位置是隐式包含在文件系统中，文件系统在某个位置必然保存了文件的`offset`。因为如果你再调用 `write` 系统调用，新写入的数据会从第 `4` 个字节开始。

在文件系统内部，文件描述符必然与某个对象关联，而这个对象不依赖文件名。操作系统内部需要对于文件有内部的表现形式，并且这种表现形式与文件名无关。

###### 文件系统

```
文件描述符
inode
inode cache
cache
disk
```

##### 专业术语

`sector` 通常是磁盘驱动可以读写的最小单元，它过去通常是 `512` 字节。

`block` 通常是操作系统或者文件系统视角的数据。它由文件系统定义。通常来说一个 `block` 对应了一个或者多个 `sector`。

##### XV6 布局节构

![img](C:\Users\tyl\Desktop\2022下半年\计网\img\assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MRhzbAZwhuzp63wWdRE%2F-MRielGcbrHOzPCrxHcO%2Fimage.png)

`block0` 要么没有用，要么被用作 `boot sector` 来启动操作系统。

`block1` 通常被称为 `super block`，它描述了文件系统。它可能包含磁盘上有多少个 `block` 共同构成了文件系统这样的信息。我们之后会看到XV6在里面会存更多的信息，你可以通过 `block1` 构造出大部分的文件系统信息。

在XV6中，`log` 从 `block2` 开始，到 `block32` 结束。实际上 `log` 的大小可能不同，这里在 `super block` 中会定义 `log` 就是 `30` 个 `block`。

接下来在 `block32` 到 `block45` 之间，XV6存储了 `inode`。我之前说过多个 `inode` 会打包存在一个 `block` 中，一个 `inode` 是 `64` 字节。

之后是 `bitmap block`，这是我们构建文件系统的默认方法，它只占据一个 `block`。它记录了数据 `block` 是否空闲。

之后就全是数据 `block` 了，数据 `block` 存储了文件的内容和目录的内容。

#### inode

一个64字节的数据结构

##### 文件结构

通常来说它有一个 `type` 字段，表明 `inode` 是文件还是目录。

`nlink` 字段，也就是 `link` 计数器，用来跟踪究竟有多少文件名指向了当前的 `inode`。

`size` 字段，表明了文件数据有多少个字节。

不同文件系统中的表达方式可能不一样，不过在XV6中接下来是一些 `block` 的编号，例如编号0，编号1，等等。XV6的 `inode` 中总共有 `12` 个`block` 编号。这些被称为 `direct block number`。这 `12` 个`block` 编号指向了构成文件的前 `12` 个 `block`。举个例子，如果文件只有 2 个字节，那么只会有一个 `block` 编号 0，它包含的数字是磁盘上文件前 2 个字节的 `block` 的位置。

之后还有一个 `indirect block number`，它对应了磁盘上一个 `block`，这个 `block` 包含了 `256` 个 `block number`，这 `256` 个 `block number` 包含了文件的数据。所以 `inode` 中 `block number 0` 到 `block number 11` 都是 `direct block number`，而 `block number 12` 保存的 `indirect block number` 指向了另一个 `block`。

<img src="C:\Users\tyl\Desktop\2022下半年\计网\img\E953D39EE3CBE32FE75817D99BECA595.jpg" alt="img" style="zoom:50%;" />

##### 目录结构

一个目录本质上是一个文件加上一些文件系统能够理解的结构。在XV6中，这里的结构极其简单。每一个目录包含了 `directory entries`，每一条 `entry` 都有固定的格式：

- 前2个字节包含了目录中文件或者子目录的 `inode` 编号，
- 接下来的14个字节包含了文件或者子目录名。

#### File system工作示例

启动XV6的过程中，调用了makefs指令，来创建一个文件系统。

- boot block
- super block
- 30个log block
- 13个inode block
- 1个bitmap block

之后是954个data block

##### 示例 echo "hi" > x

步骤

* 第一阶段是创建文件

* 第二阶段将“hi”写入文件

* 第三阶段将“\n”换行符写入到文件

![image-20231202183359475](C:\Users\tyl\Desktop\2022下半年\计网\img\image-20231202183359475.png)

##### 创建文件

1. write 33，因为 32 是 root block，所以我们写入一个新的 entry 是在 block 33 开始的，此时标记此 inode 将要被使用
2. write 33，写入 inode 的内容，包括 linknode = 1，还有其他内容
3. write 46，这是根目录的 data block，此时我们正在根目录创建一个新的文件，所以需要修改根目录下的 data block，包括新文件名，对应的 inode
4. write 32，此时根目录大小变了，需要修改 
5. write 33，此时我们再次更新文件 x 的 inode，尽管此时我们没有写数据

##### 写入 hi

1. write 45，这是更新bitmap。文件系统首先会扫描bitmap来找到一个还没有使用的data block，未被使用的data block对应bit 0。找到之后，文件系统需要将该bit设置为1，表示对应的data block已经被使用了。所以更新block 45是为了更新bitmap。

2. write 595，write 595，文件系统挑选了data block 595。所以在文件x的inode中，第一个direct block number是595。因为写入了两个字符，所以write 595被调用了两次。

3. write 33，更新文件x对应的inode中的size字段，因为现在文件x中有了两个字符。

#### `Sleep lock`

```c
void 
acquiresleeplock(struct sleeplock* lk) {
	acquire(&lk->lk);
	while (lk->locked) {
		sleep(lk, &lk->lk);
	}
	lk->locked = 1;
	lk->pid = myproc()->pid;
	release(&lk->lk);
}
```

先获取 sleep lock 关联的自旋锁，然后查看 lk 是否被 lock，如果 sleep lock 被持有，进入睡眠状态，将自己从当前 cpu 调度开，否则自己持有。

##### 为什么使用 sleep lock

1. spinlock 会关闭中断，但是中断关闭之后不能从磁盘中读取数据，同样，spinlock 的时候也不能进行 sleep
2. 我们可以在磁盘操作过程中，持有并且长期持有锁，并且等待 sleep lock 的时候，会使用 sleep 将 cpu 让出

## Lec15 Crash recovery

* 首先，它可以确保文件系统的系统调用是原子性的。比如你调用create/write系统调用，这些系统调用的效果是要么完全出现，要么完全不出现，这样就避免了一个系统调用只有部分写磁盘操作出现在磁盘上。
* 其次，它支持快速恢复（Fast Recovery）。在重启之后，我们不需要做大量的工作来修复文件系统，只需要非常小的工作量。这里的快速是相比另一个解决方案来说，在另一个解决方案中，你可能需要读取文件系统的所有block，读取inode，bitmap block，并检查文件系统是否还在一个正确的状态，再来修复。而logging可以有快速恢复的属性。

* 最后，原则上来说，它可以非常的高效，尽管我们在XV6中看到的实现不是很高效。

#### 基本思想

我们将磁盘分为两个部分，一个 log，一个文件系统

当我们写入数据的时候，每一次都是先将写操作写入 log 中

##### 步骤

1. log write，任何一次写操作都是先写入到 log，我们并不是直接写入到 block 所在的位置，而总是先将写操作写入到 log 中。
2. commit log，意味着我们需要在 log 的某个位置记录属于同一个文件系统的操作的个数
3. install log，将 block 从 log 分区移到文件系统分区。
4. clean log，清除 log 实际上就是将属于同一个文件系统的操作的个数设置为 0。

##### xv6 中的 log 结构

最开始有一个 header block，里面数字 n 代表有效 block 数量，接下来 n 个 log block 里面是 block 的编号

![img](C:\Users\tyl\Desktop\2022下半年\计网\img\419141D7F20C2D762492866987F80233.jpg)

#### log write 函数

每个文件系统操作，都有begin_op和end_op分别表示事物的开始和结束。

begin_op表明想要开始一个事务，在最后有end_op表示事务的结束。并且事务中的所有写block操作具备原子性，这意味着这些写block操作要么全写入，要么全不写入。XV6中的文件系统调用都有这样的结构，最开始是begin_op，之后是实现系统调用的代码，最后是end_op。在end_op中会实现commit操作。

在begin_op和end_op之间，磁盘上或者内存中的数据结构会更新。但是在end_op之前，并不会有实际的改变（注，也就是不会写入到实际的block中）。在end_op时，我们会将数据写入到log中，之后再写入commit record或者log header。

##### log write 步骤

先将 header block 上锁，然后查看当前写的 block 编号是否存在，存在就跳过，不存在就放在最后

数据都在对应的 cache 里面

#### end_op 函数

将 cache 里面的数据转移到 log 里面

将 log 里面的数据写入磁盘空间

## Lec17 Virtual memory for applications
