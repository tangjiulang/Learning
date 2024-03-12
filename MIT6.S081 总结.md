# MIT6.S081 2021总结

## Lab1: Xv6 and Unix utilities

#### 任务一：`Sleep`

根据实验指导书的 hint 写就好

#### 任务二：pingpong

使用两个管道的父子进程进行通信。具体是使用 pipe 进行两个进程间的通信。

这里需要注意的是 pipe 使用的是半双工通信，也就是说，不能两个进程同时进行写操作或同时进行读操作。

所以在进行通信前，我们需要确定是谁向谁发消息。

关于一个 pipe，在写进程的时候，我们需要将读的接口关闭，再读的进程，我们需要将写的接口关闭。

#### 任务三：Primes

目的是通过埃式筛求出 2~35 之间的素数。

![img](.\img\p1.png)

具体流程如上图，由于埃式筛的特性，我们每次传入右线程的第一个数字一定是一个素数，然后我们在右线程将这个素数的倍数去掉，然后除开这个数字之后，传入这个线程的右线程，然后一直递归下去，最后就能得到所有的素数

```c
void primes(int lpipe[2]) {
  close(lpipe[WR]);
  int first;
  if (lpipe_first_data(lpipe, &first) == 0) { // 提取第一个数字
    int p[2];
    pipe(p); // 当前的管道
    transmit_data(lpipe, p, first); // 将数字全部从左管道读出，然后将不能整除第一个数字的数传入右管道
    if (fork() == 0) {
      primes(p);    // 调用下一个进程进行相同的处理
    } else {
      close(p[RD]);
      wait(0);
    }
  }
  exit(0);
}
```

#### 任务四：find

看懂 ls.c 中的内容吧，然后 copy 一下

#### 任务五：xargs

字符串处理，按模拟写吧。

## Lab2: system calls

添加系统调用函数，在 `user/user.h` 中写出函数定义，在`user.pl` 中写出实体，然后去 `kernel` 中，在 `syscall.h` 中，添加系统调用号，syscall 会根据系统调用号在系统调用函数数组中找到对应的系统调用函数的地址，然后进行调用。

所以添加系统调用函数，我们需要修改 `user.h user.pl syscall.h `

#### 任务一：System call tracing

如何进行系统跟踪呢，我们每次系统调用都会进入 syscall 函数中，所以我们只需要修改 `syscall.c` 就可以对系统调用进行跟踪了

#### 任务二：Sysinfo

遍历 freelist 即可获取所有空闲内存

遍历整个 proc 数组即可获取所有进程数

## Lab3: page tables

#### 任务一：Print a page table

了解  RISC-V 页表，打印页表内容

```c
void
raw_vmprint(pagetable_t pagetable, int level){
  // there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    // PTE_V is a flag for whether the page table is valid
    if(pte & PTE_V){
      for (int j = 0; j < level; j++){
        if (j == 0) printf("..");
        else printf(" ..");
      }
      uint64 child = PTE2PA(pte);
      printf("%d: pte %p pa %p\n", i, pte, child);
      if((pte & (PTE_R|PTE_W|PTE_X)) == 0){
        // this PTE points to a lower-level page table.
        raw_vmprint((pagetable_t)child, level + 1);
      }
    }
  }
}

void
vmprint(pagetable_t pagetable){
  printf("page table %p\n", pagetable);
  raw_vmprint(pagetable, 1);
}
```

其中 raw_vmprint 可以模仿 freewalk 进行。

主要逻辑是：遍历整个页表，查看是否 valid，然后输出它指向的位置，如果指向的位置是下一级 page table，就递归查看。

#### 任务二：Detecting which pages have been accessed

本实验主要是让每个进程都有自己的内核页表，这样在内核中执行时使用它自己的内核页表的副本。

添加 PTE_A 标志位，设置最大扫描数

```c
#define PTE_A (1 << 6)
#define MAXSCAN 32
```

然后再 sysproc.c 中完成 syspgaccess 函数

具体逻辑是：

首先检查找到对应的缓冲区，然后根据是否满足 PTE_A，将掩码写入 Bitmask 中，消除 PTE_A，最后根据 copyout 复制给用户

## Lab4: Traps

#### 任务二：Backtrace

实现曾经调用函数地址的回溯

根据 hint 我们可以知道，返回地址位于栈指针（-8）的位置上，前一个帧地址保存在帧指针（-16）的位置上，所以我们只需要读出返回地址，也就是 fp - 8 的位置，然后将指针重新定位到 fp -16 的位置上就可以实现

```c
void
backtrace(void) {
  printf("backtrace:\n");
  // 读取当前帧指针
  uint64 fp = r_fp();
  while (PGROUNDUP(fp) - PGROUNDDOWN(fp) == PGSIZE) {
    // 返回地址保存在-8偏移的位置
    uint64 ret_addr = *(uint64*)(fp - 8);
    printf("%p\n", ret_addr);
    // 前一个帧指针保存在-16偏移的位置
    fp = *(uint64*)(fp - 16);
  }
}
```

XV6在内核中以页面对齐的地址为每个栈分配一个页面。使用`PGROUNDUP(fp) - PGROUNDDOWN(fp) == PGSIZE`判断当前的`fp`是否被分配了一个页面来终止循环。

#### 任务三：Alarm

实现定期的警报

要实现定期的警报，那么在定期我们就需要通过中断来进行警告。

我们先看程序计数器的工作过程

1.  `ecall` 指令中将 PC 保存到 SEPC
2. 在 `usertrap` 中将 SEPC 保存到 `p->trapframe->epc`
3. `p->trapframe->epc` 加 4 指向下一条指令
4. 执行系统调用
5. 在 `usertrapret` 中将 SEPC 改写为 `p->trapframe->epc` 中的值
6. 在 `sret` 中将 PC 设置为 SEPC 的值

也就是说，如果我们想修改回到用户空间需要进行的指令地址的话，我们需要修改 `p->trapframe->epc` ，也就是，我们需要在 usertrap 中将 `p->trapframe->epc` 设置为我们的 `alarm_hander` 函数。

但是处理完后，我们需要回到之前产生中断后一句的代码执行，所以现在我们的流程变成

1. 进入内核空间，保存用户寄存器到进程陷阱帧
2. 陷阱处理过程
3. 恢复用户寄存器，返回用户空间，但此时返回的并不是进入陷阱时的程序地址，而是处理函数 `handler` 的地址，而 `handler` 可能会改变用户寄存器
4. 恢复用户 `trapframe`，恢复陷阱帧，顺利返回

在 `proc` 中添加字段：

```c
int alarm_interval;
void (*alarm_handler)();
int ticks_count;
int is_alarming;                    // 是否正在执行告警处理函数
struct trapframe* alarm_trapframe;  // 告警陷阱帧
```

在 sys_sigalarm 中读取参数

```c
uint64
sys_sigalarm(void) {
  if(argint(0, &myproc()->alarm_interval) < 0 ||
    argaddr(1, (uint64*)&myproc()->alarm_handler) < 0)
    return -1;
  return 0;
}
```

修改 usertrap

```c
if(which_dev == 2) {
  if(p->alarm_interval != 0 && ++p->ticks_count == p->alarm_interval && p->is_alarming == 0) {
    // 保存寄存器内容
    memmove(p->alarm_trapframe, p->trapframe, sizeof(struct trapframe));
    // 更改陷阱帧中保留的程序计数器，注意一定要在保存寄存器内容后再设置epc
    p->trapframe->epc = (uint64)p->alarm_handler;
    p->ticks_count = 0;
    p->is_alarming = 1;
  }
  yield();
}
```

增添对 proc 中增添字段的初始化和回收

最后修改 sys_sigreturn，恢复陷阱帧

```c
uint64
sys_sigreturn(void) {
  memmove(myproc()->trapframe, myproc()->alarm_trapframe, sizeof(struct trapframe));
  myproc()->is_alarming = 0;
  return 0;
}
```

## Lab: Copy-on-Write Fork for xv6

`fork()` 将父进程的所有用户空间内存复制到子进程，如果父进程比较大，复制需要很长时间。

并且，在很多时候会造成资源浪费，比如，子进程只对其中几个页面进行修改，如果复制完整的内存，很多内存都被浪费。

#### copy-on-write

将复制的过程推迟到 pagefault 的时候，在 fork 的时候，我们让父子进程都指向同一个内存，只有当子进程需要的时候，分配页面给子进程

##### 步骤：

1. 添加 PTE_F 来标志这是不是 cow fork 的标志位
2. 添加引用计数的全局变量 ref

```c
struct {
  struct spinlock lock;
  int cnt[PHYSTOP / PGSIZE]
} ref;
```

​	然后在 init 中初始化 ref

```c
void
kinit()
{
  initlock(&kmem.lock, "kmem");
  initlock(&ref.lock, "ref");
  freerange(end, (void*)PHYSTOP);
}
void
freerange(void *pa_start, void *pa_end)
{
  char *p;
  p = (char*)PGROUNDUP((uint64)pa_start);
  for(; p + PGSIZE <= (char*)pa_end; p += PGSIZE) {
    // 在kfree中将会对cnt[]减1，这里要先设为1，否则就会减成负数
    ref.cnt[(uint64)p / PGSIZE] = 1;
    kfree(p);
  }
}
```

​	修改 kalloc 还有 kfree 函数，kalloc 中初始化引用为 1，kfree 中将内存引用计数减 1，知道引用计数为 0 的时候，才真正的删除

```c
void *
kalloc(void) {
  struct run *r;

  acquire(&kmem.lock);
  r = kmem.freelist;
  if(r) {
    kmem.freelist = r->next;
    acquire(&ref.lock);
    ref.cnt[(uint64)r / PGSIZE] = 1;  // 将引用计数初始化为1
    release(&ref.lock);
  }
  release(&kmem.lock);

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}
```

```c
void
kfree(void *pa) {
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // 只有当引用计数为0了才回收空间
  // 否则只是将引用计数减1
  acquire(&ref.lock);
  if(--ref.cnt[(uint64)pa / PGSIZE] == 0) {
    release(&ref.lock);

    r = (struct run*)pa;

    // Fill with junk to catch dangling refs.
    memset(pa, 1, PGSIZE);

    acquire(&kmem.lock);
    r->next = kmem.freelist;
    kmem.freelist = r;
    release(&kmem.lock);
  } else {
    release(&ref.lock);
  }
}
```

  在 kfree 中需要注意的一点是，应该将引用计数减一和读取减一后的引用计数放在一起，如果只将 -- 操作放在 lock 内，那么如果两个进程对这个页面同时进行 -- 操作，那么读到的数字就不是我们想要的数字

​	编写 cowpage 函数，检查这个 pagefault 是否是 cowfork 引起的

```c
int cowpage(pagetable_t pagetable, uint64 va) {
  if(va >= MAXVA)
    return -1;
  pte_t* pte = walk(pagetable, va, 0);
  if(pte == 0)
    return -1;
  if((*pte & PTE_V) == 0)
    return -1;
  return (*pte & PTE_F ? 0 : -1);
}
```

根据 hint 进行 cowpage 的处理，具体流程看代码所示

```c
void* cowalloc(pagetable_t pagetable, uint64 va) {
  if(va % PGSIZE != 0)
    return 0;
// 获取对应的物理地址
  uint64 pa = walkaddr(pagetable, va);
  if(pa == 0)
    return 0;
// 获取对应的PTE
  pte_t* pte = walk(pagetable, va, 0);  

  if(krefcnt((char*)pa) == 1) {
    // 直接修改对应的PTE即可
    *pte |= PTE_W;
    *pte &= ~PTE_F;
    return (void*)pa;
  } else {
    // 需要分配新的页面，并拷贝旧页面的内容
    char* mem = kalloc();
    if(mem == 0)
      return 0;
    // 复制旧页面内容到新页
    memmove(mem, (char*)pa, PGSIZE);
    // 清除PTE_V，否则在mappagges中会判定为remap
    *pte &= ~PTE_V;
    // 为新页面添加映射
    if(mappages(pagetable, va, PGSIZE, (uint64)mem, (PTE_FLAGS(*pte) | PTE_W) & ~PTE_F) != 0) {
      kfree(mem);
      *pte |= PTE_V;
      return 0;
    }
    // 将原来的物理内存引用计数减1
    kfree((char*)PGROUNDDOWN(pa));
    return mem;
  }
}
```

添加两个辅助函数，kaddrefcnt 和 krefcnt

```c
int krefcnt(void* pa) {
  return ref.cnt[(uint64)pa / PGSIZE];
}
int kaddrefcnt(void* pa) {
  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    return -1;
  acquire(&ref.lock);
  ++ref.cnt[(uint64)pa / PGSIZE];
  release(&ref.lock);
  return 0;
}
```

3. 修改 uvmcopy，在 fork 的时候不为子进程分配内存，而是和父进程共享，并且禁用 PTE_W，开启 PTE_F，同时调用 kaddrefcnt 增加引用计数

```c
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);

    // 仅对可写页面设置COW标记
    if(flags & PTE_W) {
      // 禁用写并设置COW Fork标记
      flags = (flags | PTE_F) & ~PTE_W;
      *pte = PA2PTE(pa) | flags;
    }

    if(mappages(new, i, PGSIZE, pa, flags) != 0) {
      uvmunmap(new, 0, i / PGSIZE, 1);
      return -1;
    }
    // 增加内存的引用计数
    kaddrefcnt((char*)pa);
  }
  return 0;
}
```

4. 修改 usertrap 当 pagefault 是 cowpage，就调用 cowalloc

```c
else if(cause == 13 || cause == 15) {
  uint64 fault_va = r_stval();  // 获取出错的虚拟地址
  if(fault_va >= p->sz
    || cowpage(p->pagetable, fault_va) != 0
    || cowalloc(p->pagetable, PGROUNDDOWN(fault_va)) == 0)
    p->killed = 1;
}
```

4. 修改 copyout 支持 cowpage

```c
while(len > 0){
  va0 = PGROUNDDOWN(dstva);
  pa0 = walkaddr(pagetable, va0);
  // 处理COW页面的情况
  if(cowpage(pagetable, va0) == 0) {
    // 更换目标物理地址
    pa0 = (uint64)cowalloc(pagetable, va0);
  }
  if(pa0 == 0)
    return -1;
  ...
}
```

## Lab7: Multithreading

本实验主要体验用户级线程系统上下文切换机制，主要修改 user/uthread.c 和 user/uthread_switch.S

可以参考 kernel 里面的 proc 里面的 context，定义我们自己的 context，只定义 context 是因为我们只需要在用户级进行切换，不需要完整的 trapframe，然后加入到 thread 中，然后参考 kernel/switch.S 编写 user/uthread_switch.S

```c
.globl thread_switch
thread_switch:
    /* YOUR CODE HERE */
    sd ra, 0(a0)
    sd sp, 8(a0)
    sd s0, 16(a0)
    sd s1, 24(a0)
    sd s2, 32(a0)
    sd s3, 40(a0)
    sd s4, 48(a0)
    sd s5, 56(a0)
    sd s6, 64(a0)
    sd s7, 72(a0)
    sd s8, 80(a0)
    sd s9, 88(a0)
    sd s10, 96(a0)
    sd s11, 104(a0)

    ld ra, 0(a1)
    ld sp, 8(a1)
    ld s0, 16(a1)
    ld s1, 24(a1)
    ld s2, 32(a1)
    ld s3, 40(a1)
    ld s4, 48(a1)
    ld s5, 56(a1)
    ld s6, 64(a1)
    ld s7, 72(a1)
    ld s8, 80(a1)
    ld s9, 88(a1)
    ld s10, 96(a1)
    ld s11, 104(a1)
    ret    /* return to ra */
```

修改 thread_create 进行初始化设定，主要是修改 ra 返回地址和 sp 栈地址

```c
t->context.ra = (uint64)func;                   // 设定函数返回地址
t->context.sp = (uint64)t->stack + STACK_SIZE;  // 设定栈指针
```

在 thread_scheduler 中，添加线程切换语句

```c
if (current_thread != next_thread) {         /* switch threads?  */
  ...
  /* YOUR CODE HERE */
  thread_switch((uint64)&t->context, (uint64)&current_thread->context);
} else
  next_thread = 0;
```

剩下两个任务主要是在 linux 下进行 thread 的体验

##  Lab8: Locks

#### 任务一：Memory allocator

三个进程增长和缩小地址空间，导致对`kalloc`和`kfree`的多次调用。`kalloc`和`kfree`获得`kmem.lock`。`kalloctest`打印在`acquire`中由于尝试获取另一个内核已经持有的锁而进行的循环迭代次数，如`kmem`锁和一些其他锁。`acquire`中的循环迭代次数是锁争用的粗略度量。

`kalloctest` 中锁争用的根本原因是 `kalloc()` 有一个空闲列表，由一个锁保护。要消除锁争用，您必须重新设计内存分配器，以避免使用单个锁和列表。基本思想是为每个 CPU 维护一个空闲列表，每个列表都有自己的锁。因为每个 CPU 将在不同的列表上运行，不同 CPU 上的分配和释放可以并行运行。主要的挑战将是处理一个 CPU 的空闲列表为空，而另一个 CPU 的列表有空闲内存的情况；在这种情况下，一个 CPU 必须“窃取”另一个 CPU 空闲列表的一部分。窃取可能会引入锁争用，但这种情况希望不会经常发生。

您的工作是实现每个CPU的空闲列表，并在CPU的空闲列表为空时进行窃取。

通过上面的提示，我们可以对每一个 cpu 都分配一个 freelist，每次够的时候从其他 cpu 中窃取一个。

那么我们就可以将 kmem 修改为

```c
struct kmem {
	spinlock lock;
	struct run* freelist;
};
strct kmem kmems[NCPU]
```

然后修改 init 将 kmem 进行初始化

```c
void
init() {
	for (int i = 0; i < NCPU; i++) {
		char name[9] = {0};
		snprintf(name, 8, "kmem_%d", i);
		initlock(&(kmems[i].lock), name);
	}
	freerange(end, (void*)PHYSTOP);
}
```

然后我们需要修改 kalloc 和 kfree 来支持

```c
void *
kalloc(void) {
	struct run* r;
	push_off();
	int cpuid = cpuid();
	acquire(&kmems[cpuid].lock);
	r = kmems[cpuid].freelist;
	if (r) {
		kmems[cpuid].freelist = r->next;
	}
	release(&kmems[cpuid].lock);
	if (r == 0) {
		r = ksteal(cpuid);
	}
	if (r)
		memset((char*)r, 5 PGSIZE);
	pop_off();
	return (void*)r
}
void *
kfree(void* pa) {
    ...
    push_off();
    int cpuid = cpuid();
    memset(pa, 1, PGSIZE);
    
    r = (struct run*) pa;
    
    acquire(&kmems[cpuid].lock);
    r->next = kmems[cpuid].freelist;
    kmems[cpuid].freelist = r;
    release(&kmems[cpuid].lock);
    
    pop_off();
}
```

这里有一个问题，为什么要用 push_off() 来关闭中断，在 hint 里面，它说的是只有在中断关闭时调用它并使用其结果才是安全的。那么我们可以知道在申请内存时需要通过 cpuid() 获取 CPU 核编号 hartid，这个函数是没有锁保护的，所以获取 cpuid 时需要关中断，避免被切换出去而破坏了临界区。简单来说就是怕被其他中断程序破坏了寄存器，得到错误的值。

然后编写 ksteal 函数

```c
void *
ksteal(int cpuid) {
	struct run* r;
	for (int i = 1; i < NCPU; i++) {
		int id = (cpuid + i) % NCPU;
         acquire(&kmems[id].lock);
         r = kmems[id].freelist;
         if (r)
        	kmem[id].freelist = r.next;
         release(&kmems[id].lock);
         if (r)
         	break;
	}
	return (void*) r;
}
```

#### 任务二：Buffer cache

我们现在有一个

```c
struct {
	struct spinlock lock;
	struct buf buf[NBUF];
	struct buf head;
} bcache;
```

如果进程频繁使用系统，那么他们可能会争夺 bcache.lock，我们的目的是减少 bcache 中所有锁的争用，根据 hint，我们可以使用 hash bucket 的方式来减少争用，简单来说就是将一个顺序结构 bcache 变成多个 hash 桶结合在一起的一个结构。

```c
struct bcache_bucket {
	struct spinlock lock;
	struct buf head;
};

struct {
	struct spinlock lock;
	struct buf buf[NBUF];
	struct bcache_bucket bucket[NBUCKET];
} bcache;

int
hash(uint blockno) {
    return blockno % NBUCKET;
}
```

由于是 lru，所以我们需要当前 buf 的 ticks，所以我们要在 buf 里面添加字段 access_time 来保存 ticks。

修改 binit 函数，对 bcache 进行初始化

```c
void
binit(void) {
	char name[32];
	int sz = 32;
	initlock(&bcache.lock, "bcache");
	for (int i = 0; i < NBUCKET; i++) {
		snprintf(name, sz, "bcache.bucket_%d", i);
		initlock(&bcache.bucket[i].lock, name);
	}
	int blockcnt = 0;
	struct bcache_bucket* bucket;
	struct buf* b;
	for (b = bcache.buf; b < bcache.buf + NBUF; b++) {
		initsleeplock(&b.lock, "buffer");
		b.access_time = ticks;
		b.blockno = blockcnt++;
		bucket = bcache.bucket[hash(blockcnt)];
		b->next = bucket->head.next;
		bucket->head.next = b;
	}
}
```

修改 bget，使其能够找到合适的 buf

```c
static struct buf*
bget(uint dev, uint blockno)
{
  struct buf *b, *lrub;
  struct bcache_bucket* bucket = &bcache.bucket[hash(blockno)];
  acquire(&bucket->lock);

// find in bucket is cached
  for (b = &bucket->head; b; b = b->next) {
    if (b->dev == dev && b->blockno == blockno) {
      b->refcnt++;
      b->access_time = ticks;
      release(&bucket->lock);
      acquiresleep(&b->lock);
      return b;
    }
  }

// no cached
// find in lru bucket
  lrub = 0;
  uint least = 0xffffffff;
  for (b = &bucket->head; b; b = b->next) {
    if (b->refcnt == 0 && b->access_time < least) {
      lrub = b;
      least = b->access_time;
    }
  }
  if (lrub) {
    lrub->dev = dev;
    lrub->blockno = blockno;
    lrub->valid = 0;
    lrub->refcnt = 1;
    lrub->access_time = ticks;
    release(&bucket->lock);
    acquiresleep(&lrub->lock);
    return lrub;
  }
// steal from other bucket
  acquire(&bcache.lock);
findbucket:
  lrub = 0;
  uint min_time = 0x8ffffff;
  for(b = bcache.buf; b < bcache.buf+NBUF; b++){
    if(b->refcnt == 0 && b->access_time < min_time) {
      lrub = b;
    }
  }
 
  if (lrub) {
    // step 1 : release from the old bucket
    // need to hold the old bucket lock
    struct bcache_bucket* old_bucket = &bcache.bucket[hash(lrub->blockno)];
    acquire(&old_bucket->lock);
    
    if (lrub->refcnt != 0){
      release(&old_bucket->lock);
      goto findbucket;
    }
 
    for (b = &old_bucket->head; b; b = b->next) {
      if (b->next == lrub) {
        b->next = lrub->next;
        break;
      }
    }
    // we don't need to modify bcache.bucket , so we release the lock
    release(&old_bucket->lock);
    // step 2 : add to target bucket 
    lrub->next = bucket->head.next;
    bucket->head.next = lrub;
    release(&bcache.lock);
    lrub->dev = dev;
    lrub->blockno = blockno;
    lrub->valid = 0;
    lrub->refcnt = 1;
    lrub->access_time = ticks;
    release(&bucket->lock);
    acquiresleep(&lrub->lock);
    return lrub;
  }
  panic("bget: no buffers");
}
```

然后修改 brelse

```c
void
brelse(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("brelse");

  releasesleep(&b->lock);

  int id = hash(b->blockno);
  acquire(&bcache.bucket[id].lock);
  b->refcnt--;
  release(&bcache.bucket[id].lock); 
}
```

bunpin 和 bpin 进行同样的修改，使其变成原子操作

这里顺便介绍一下什么是 sleeplock

当访问共享页面的时候，进程首先会尝试获取 sleeplock 如果 sleeplock 已被占用，那么进程主动陷入沉睡，释放 cpu 的所有权，如果其他线程释放了锁，那么唤醒进程，重新尝试获取锁并继续运行

## Lab9: file system

#### 任务一： Large files

模仿之前的 dinode 的二级结构，将 dinode 改为三级结构。（模仿之前的已有函数即可）

```c
#define NDIRECT 11
#define NNDIRECT NDIRECT + 1
#define NINDIRECT (BSIZE / sizeof(uint))
#define MAXFILE (NDIRECT + NINDIRECT + NINDIRECT * NINDIRECT)

// On-disk inode structure
struct dinode {
  short type;           // File type
  short major;          // Major device number (T_DEVICE only)
  short minor;          // Minor device number (T_DEVICE only)
  short nlink;          // Number of links to inode in file system
  uint size;            // Size of file (bytes)
  uint addrs[NNDIRECT + 1];   // Data block addresses
};
static uint
bmap(struct inode *ip, uint bn)
{
  uint addr, *a, *b;
  struct buf *bp, *cp;

  if(bn < NDIRECT){
    if((addr = ip->addrs[bn]) == 0)
      ip->addrs[bn] = addr = balloc(ip->dev);
    return addr;
  }
  bn -= NDIRECT;

  if(bn < NINDIRECT){
    // Load indirect block, allocating if necessary.
    if((addr = ip->addrs[NDIRECT]) == 0)
      ip->addrs[NDIRECT] = addr = balloc(ip->dev);
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if((addr = a[bn]) == 0){
      a[bn] = addr = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp);
    return addr;
  }
  bn -= NINDIRECT;
  
  if (bn < NINDIRECT * NINDIRECT) {
    // Load inndirect block, allocating if necessary.
    if ((addr = ip->addrs[NNDIRECT]) == 0)
      ip->addrs[NNDIRECT] = addr = balloc(ip->dev);
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;

    uint t = bn / NINDIRECT;
    if ((addr = a[t]) == 0) {
      a[t] = addr = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp);
    cp = bread(ip->dev, addr);
    b = (uint*)cp->data;
    t = bn % NINDIRECT;
    if ((addr = b[t]) == 0) {
      b[t] = addr = balloc(ip->dev);
      log_write(cp);
    }
    
    brelse(cp);
    return addr;
  }
  panic("bmap: out of range");
}

// Truncate inode (discard contents).
// Caller must hold ip->lock.
void
itrunc(struct inode *ip)
{
  int i, j;
  struct buf *bp, *cp;
  uint *a, *b;

  for(i = 0; i < NDIRECT; i++){
    if(ip->addrs[i]){
      bfree(ip->dev, ip->addrs[i]);
      ip->addrs[i] = 0;
    }
  }

  if(ip->addrs[NDIRECT]){
    bp = bread(ip->dev, ip->addrs[NDIRECT]);
    a = (uint*)bp->data;
    for(j = 0; j < NINDIRECT; j++){
      if(a[j])
        bfree(ip->dev, a[j]);
    }
    brelse(bp);
    bfree(ip->dev, ip->addrs[NDIRECT]);
    ip->addrs[NDIRECT] = 0;
  }

  if (ip->addrs[NNDIRECT]) {
    bp = bread(ip->dev, ip->addrs[NNDIRECT]);
    a = (uint*)bp->data;
    for (i = 0; i < NINDIRECT; i++) {
      if (a[i]) {
        cp = bread(ip->dev, a[i]);
        b = (uint*)cp->data;
        for (j = 0; j < NINDIRECT; j++) {
          if (b[j])
            bfree(ip->dev, b[j]);
        }
        brelse(cp);
        bfree(ip->dev, a[i]);
        a[i] = 0;  
      }
    }
    brelse(bp);
    bfree(ip->dev, ip->addrs[NNDIRECT]);
    ip->addrs[NNDIRECT] = 0;
  }
  ip->size = 0;
  iupdate(ip);
}
```

#### 任务二：Symbolic links

实现 `symlink(char *target, char *path)` 系统调用

实现系统调用之前的步骤在上文已经介绍过了，这里主要讲 symlink

首先我们先介绍一下什么是 symlink，符号链接是由一个路径指向另一个路径的文件或目录。当通过符号链接访问文件或目录时，操作系统会将请求重定向到符号链接所指向的实际目标。在文件系统中，符号链接是一个包含目标路径的文件，而不是实际的文件或目录。

根据 hint 添加 NOFOLLOW 字段还有 T_SYMLINK 类型至相关文件，然后修改 sys_open 使其支持符号链接

```c
uint64
sys_open(void) {
	...
	
// be sure the ip is symlink and omode is follow
  if (ip->type == T_SYMLINK && !(omode & O_NOFOLLOW)) {
// for the max depth
    for (int i = 0; i < MAXDEPTH; i++) {
// read old path in symlink file
      if (readi(ip, 0, (uint64)path, 0, MAXPATH) <= 0) {
        iunlockput(ip);
        end_op();
        return -1;
      } 
// unneed the old inode
      iunlockput(ip);
// find new inode from path
      if ((ip = namei(path)) == 0) {
        end_op();
        return -1;
      }

      ilock(ip);
      if (ip->type != T_SYMLINK)
        break;
    }

    if (ip->type == T_SYMLINK) {
      iunlockput(ip);
      end_op();
      return -1;
    }
  }
  
  ...
}
```

编写 sys_symlink 函数

```c
// 0 old
// 1 new
uint64
sys_symlink(void) {
  char old[MAXPATH], new[MAXPATH];
  if (argstr(0, old, MAXPATH) < 0) 
    return -1;
  if (argstr(1, new, MAXPATH) < 0)
    return -1;
    
  begin_op();
  struct inode* ip;
  // get a new symlink
  if ((ip = create(new, T_SYMLINK, 0, 0)) == 0) {
    end_op();
    return -1;
  }
  // build link (new symlink) to (old path)
  if (writei(ip, 0, (uint64)old, 0, MAXPATH) < MAXPATH) {
    iunlockput(ip);
    end_op();
    return -1;
  }
  iunlockput(ip);
  end_op();
  return 0;

}
```

这里有几点需要知道 begin_op() 和 end_op() 是启动和关闭文件系统，create 中返回的是加锁的 inode，所以退出的时候会使用 iunlockput 进行解锁，并且讲引用计数 -1，如果引用计数为 0 就会收回 inode

## Lab10: mmap

实现一个内存映射文件的功能，将文件映射到内存中，从而在与文件交互时减少磁盘操作。

利用了 lazy page allocation 的思想，在 mmap 的时候不分配内存和读取文件，当发生 pagefault 的时候，调用 usertrap 中的 pagefault handler 来解决

添加 vma 至 proc.h

```c
struct vma {
  uint64 addr;         // 起始地址
  uint64 length;       // 映射长度
  uint flags;
  uint prot;           // shared 还是 private
  struct file* file;   // 映射文件
  uint8 valid;
  uint64 dirtyflag;    // 是否脏
};

struct proc {
	...
	struct vma vma[16];
};
```

添加 getvma 函数，根据虚拟地址 va 得到对应的 vma

```c
struct vma* getvma(uint64 va) {
  struct proc* proc = myproc();
  struct vma* vma = 0;
  for (int i = 0; i < 16; i++) {
    if (proc->vma[i].valid == 0) 
      continue;
    uint64 addr = proc->vma[i].addr;
    uint64 length = proc->vma[i].length;
    if (addr <= va && (va < addr + length)) {
      vma = &proc->vma[i];
      break;
    }
  }
  return vma;
}
```

修改 usertrap，将 pagefault 转到 pagefault handler 进行处理

```c
  else if (r_scause() == 15 || r_scause() == 13) {
// page fault
// the virtual memory of pagefault
    uint64 va = r_stval();
    if (va > MAXVA || (va <= PGROUNDDOWN(p->trapframe->sp) && va >= PGROUNDDOWN(p->trapframe->sp) - PGSIZE)) {
      p->killed = 1;
    } else {
      if (pagefault(p->pagetable, va) < 0)
        p->killed = 1;
    }
  } 
```

编写 mmap 函数，使其将当前 `p->sz` 作为分配的虚拟起始地址，但不实际分配物理页面

```c
uint64
sys_mmap(void) {
  uint64 addr;
  int length, prot, flags, offset;
  struct file* file = 0;
  if (argaddr(0, &addr) < 0)
    return -1;
  if (argint(1, &length) < 0)
    return -1;
  if (argint(2, &prot) < 0)
    return -1;
  if (argint(3, &flags) < 0)
    return -1;
  if (argfd(4, 0, &file) < 0)
    return -1;
  if (argint(5, &offset) < 0)
    return -1;
// check permission
  if (!file->readable && (prot & PROT_READ))
    return -1;
  if (!file->writable && (prot & PROT_WRITE) && !(flags & MAP_PRIVATE))
    return -1;
  struct proc* proc = myproc();
  struct vma* vma = 0;
// find a new vma
  for (int i = 0; i < 16; i++) {
    if (proc->vma[i].addr == 0) {
      vma = &proc->vma[i];
      break;
    }
  }

  if (vma == 0)
    return -1;
  vma->length = length;
  vma->prot = prot;
  vma->flags = flags;
  vma->file = file;
  vma->valid = 1;

// add heap size
// lazy page allocation
  uint64 retaddr = PGROUNDUP(proc->sz);
  proc->sz = retaddr + length;
  vma->addr = retaddr;

// add file ref cnt
  filedup(vma->file);

// record which page be mapped
  int pgcnt = length / PGSIZE;
  int mask = 1;
  for (int i = 0; i < pgcnt; i++) {
    vma->dirtyflag |= (mask << i);
  }

  return retaddr;
}
```

mmap 没有实际分配内存，在发生 pagefault 后，usertrap 找到了 pagefault 函数

```c
int
pagefault(pagetable_t pagetable, uint64 fault_va) {
// check addr in vma
  struct vma* vma = getvma(fault_va);
  if (vma == 0)
    return -1;
// check permission
  if (r_scause() == 13 && (!(vma->prot & PROT_READ) || !(vma->file->readable)))
    return -1;
  if (r_scause() == 15 && (!(vma->prot & PROT_WRITE) || !(vma->file->writable)))
    return -1;

// kalloc new page and map it, setup permission flag
// kalloc new page
  void* dst_pa = kalloc();
  if (dst_pa == 0)
    return -1;

// setup permission flag
  uint8 flag = (vma->prot & PROT_READ) ? PTE_R : 0;
  flag |= (vma->prot & PROT_WRITE) ? PTE_W : 0;

// map new page
  if (mappages(pagetable, PGROUNDDOWN(fault_va), PGSIZE, (uint64)dst_pa, PTE_U | PTE_X | flag) < 0) {
    kfree(dst_pa);
    return -1;
  }

// load file content to memory
  uint64 offset = PGROUNDDOWN(fault_va) - vma->addr;
  vma->file->off = offset;

  int read = 0;
  if ((read = fileread(vma->file, PGROUNDDOWN(fault_va), PGSIZE)) < 0)
    return -1;
  
// should clear zero
  if (read < PGSIZE) {
    uint64 pa = walkaddr(pagetable, PGROUNDDOWN(fault_va)) + read;
    memset((void*)pa, 0, PGSIZE - read);
  }
  return 0;
}
```

实现 munmap 函数，将 shared 写回文件系统

```c
uint64
sys_munmap(void) {
  uint64 addr;
  int length;
  if (argaddr(0, &addr) < 0)
    return -1;
  if (argint(1, &length) < 0)
    return -1;
  struct vma* vma = getvma(addr);
  if (vma == 0)
    return -1;

  if (length > vma->length || addr < vma->addr)
    return -1;

  int start = (addr - vma->addr) / PGSIZE;
  int end = start + (length % PGSIZE == 0) ? (length / PGSIZE) : (length / PGSIZE + 1);

  if (vma->flags & MAP_PRIVATE) {
    goto finish;
  } else if (vma->flags & MAP_SHARED) {
    for (int i = start; i < end; i++) {
      if (walkaddr(myproc()->pagetable, (vma->addr + i * PGSIZE))) {
        vma->file->off = PGSIZE * i;
        filewrite(vma->file, vma->addr, PGSIZE);
      }
    }
  }
  uint mask = 1;

finish:
  for (int i = start; i < end; i++) {
    vma->dirtyflag &= ~(mask << i);
  }

  printf("dirtyflag: %d\n", vma->dirtyflag);

  if (vma->dirtyflag == 0) {
    printf("all area unmmap, start recyle\n");
    vma->valid = 0;
    fileclose(vma->file);
  }

  return 0;
}
```

同时，由于如果对惰性分配的页面调用了 `uvmunmap`，或者子进程在 fork 中调用 `uvmcopy` 复制了父进程惰性分配的页面都会导致 panic，因此需要修改 `uvmunmap` 和 `uvmcopy` 检查 `PTE_V` 后不再 `panic`

```c
if((*pte & PTE_V) == 0)
      continue;
```

修改 fork，复制父进程的VMA并增加文件引用计数

```c
fork() {
  ...
  for (int i = 0; i < 16; i++) {
    if (p->vma[i].valid) {
      np->vma[i].addr = p->vma[i].addr;
      np->vma[i].dirtyflag = p->vma[i].dirtyflag;
      np->vma[i].file = p->vma[i].file;
      np->vma[i].flags = p->vma[i].flags;
      np->vma[i].length = p->vma[i].length;
      np->vma[i].prot = p->vma[i].prot;
      np->vma[i].valid = 1;
      filedup(np->vma[i].file);
    }
  }
  
  ...
}
```

修改 exit，将映射解除

```c
exit() {
...
 for(int i = 0; i < 16; ++i) {
    if(p->vma[i].valid) {
      if(p->vma[i].flags == MAP_SHARED && (p->vma[i].prot & PROT_WRITE) != 0) {
        filewrite(p->vma[i].file, p->vma[i].addr, p->vma[i].length);
      }
      fileclose(np->vma[i].file);
      uvmunmap(p->pagetable, p->vma[i].addr, p->vma[i].length / PGSIZE, 1);
      p->vma[i].used = 0;
    }
  }
}
```

