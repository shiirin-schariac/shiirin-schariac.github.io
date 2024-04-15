有3种进入trap的情况：

- 系统调用
- 异常
- 中断

一旦代码在执行时发生了trap，会按照如下步骤进行对trap的处理：trap迫使控制权转移到内核 -> **内核**保存寄存器和其他状态，以便恢复执行 -> 内核执行适当的处理程序代码（例如，系统调用实现或设备驱动程序） -> 内核恢复保存的状态，并从trap中返回 -> 代码从原来的地方恢复执行。

以上的部分可以简化为4个阶段：RISCV CPU的硬件行为 -> 汇编指令准备一个入口进入内核C代码 -> 运行内核中处理trap的C程序 -> 系统调用或设备驱动服务

### 4-1 RISCV的硬件行为
关于几个关键的寄存器：

- `stvec` : 内核将trap handler（例如在用户空间下的trap，是trampoline的 `uservec` ）写到 `stvec` 中（？？为何是内核写进，一开始处于用户态内核是如何写进的？已破案详见下文）。当trap发生时，RISCV就会跳转到 `stvec` 中的地址，准备处理trap。我们可以将trap vector认为是trap handler的准备工作部分。

- `sepc` : trap发生时，RISCV会将pc原本储存的地址放在这里，当从trap中返回时， `sret` 会将 `sepc` 中放到pc中以恢复程序的执行。

- `sscratch`：一个特别的寄存器，通常在用户空间发生trap，并进入到uservec之后，sscratch就装载着指向进程**trapframe**的指针（该进程的trapframe，在进程被创建，并从userret返回的时候，就已经被内核设置好并且放置到sscratch中）<- ==这里需要联系下面的 `userret` 部分==。RISC-V还提供了一条**交换指令**（csrrw），可以将任意寄存器与sscratch进行值的交换。sscratch的这些特性，便于在uservec中进行一些寄存器的保存、恢复工作。

- `sstatus`：`sstatus`中的**SIE**位控制设备中断是否被启用，如果内核清除**SIE**，RISC-V将推迟设备中断，直到内核设置**SIE**。**SPP**位表示trap是来自用户模式还是supervisor模式，并控制`sret`返回到什么模式。

每个CPU都有一套单独的状态处理器，以便处理多个trap。

RISCV**硬件**处理trap的步骤：
（注意！全部都是硬件做的工作）

1. 如果该trap是设备中断，且`sstatus` **SIE**位为0（说明该中断为硬件中断），则不执行以下任何操作。
2. 通过清除SIE来禁用中断。
3. 复制`pc`到`sepc`。
4. 将当前模式（用户态或特权态）保存在`sstatus`的**SPP**位。
5. 在`scause`设置该次trap的原因。
6. 将模式转换为特权态。（但这里没有进入内核）
7. 将`stvec`复制到`pc`。
8. 从新的`pc`开始执行。

在该阶段硬件所做工作极少：CPU的 `satp` 不会切换到内核页表，使用的栈不会切换至内核栈，也不保存除pc以外的任何寄存器。

> 疑问：究竟是谁把 `uservec` 写进 `stvec` 的……
> 
> 回答：看了一下似乎是上一次trap结束后由 `usertrapret` 放进去的，那么调用 `kernelvec` 的时候为什么 `stvec` 的起始值为 `kernelvec` ? -> 是进入内核时会这样初始化一下，对应代码在 `usertrap` 中

### 4-2 用户空间的trap
前置知识： `trapframe` 是一页位于trampoline页下的页表。进程的`p->trapframe`也指向`trapframe`，不过是指向它的物理地址，这样内核可以通过内核页表来使用它。

用户空间的trap执行代码的路径是： `uservec` -> `usertrap` -> `usertrapret` -> `userret` 

#### uservec（用户态）
`uservec` ：其地址被放在trampoline页中（使每一个用户进程都可以通过trampoline运行 `uservec`）， `stvec` 首先被设置为 `uservec` 以开始运行 `uservec` 。将 `trapframe` 的地址放在寄存器 `sscratch` 中，通过 `csrw` 指令将其与寄存器 `a0` 内容互换后，`a0`将指向当前进程的`trapframe`。`uservec`将在`trapframe`保存全部的寄存器，包括从`sscratch`读取的`a0`。`trapframe`包含指向当前进程的内核栈、当前CPU的hartid、`usertrap`的地址和内核页表的地址的指针，`uservec`将这些值设置到相应的寄存器中，并将`satp`切换到内核页表和刷新TLB，然后调用`usertrap`。

> 对uservec的总结：
> 	1. 将用户进程的寄存器保存到 trapframe 中
> 	2. 切换内核栈，内核线程 id，内核页表
> 	3. 跳转到 kernel/trap.c 的 `usertrap` 函数

#### usertrap（用户态 -> 内核态）
`usertrap` : 其作用是确定trap的原因，处理它（**进入内核处理**），然后返回。 `stvec` 首先被设置为 `kernelvec` 以开始处理trap（这一步在进入内核时完成）。在 `usertrap` 里对 `sepc` 进行保存（用于下一步的恢复）。接下来 `usertrap` 根据实际情况处理trap，在退出时， `usertrap` 检查进程是否已经被杀死或应该让出CPU（如果这个trap是一个定时器中断）。

> 总结：
> 	1. 把 `stvec` 改成 `kernelvec` 以处理发生在内核的 trap
> 	2. 保存用户进程的 pc 到 trapframe
> 	3. 根据 `scause`、`devintr` 处理 trap
> 	4. 检查是否计时器中断，若是则进行进程调度
> 	5. 检查进程是否被杀死，若是则退出
> 	6. 跳转到 `usertrapret` 函数

#### usertrapret（内核态 -> 用户态）
`usertrapret` : 回到用户空间的第一步是调用 `usertrapret` 。这个函数设置RISC-V控制寄存器，为以后用户空间trap做准备。这包括改变 `stvec` 来引用 `uservec` ，准备 `uservec` 所依赖的 `trapframe` 字段，并将 `sepc` 设置为先前保存的用户程序计数器。最后， `usertrapret` 在用户页表和内核页表中映射的trampoline页上调用 `userret` 并传入参数参数 `a0` 与 `a1` ，其中 `a0` 指向 `TRAPFRAME` ， `a1` 指向用户进程页表，使用 `userret` 中的汇编代码切换页表。

> 总结：
> 	1. 保存内核页表、内核栈、CPU 核的 id
> 	2. 配置 sstatus 寄存器
> 	3. 调用 `userret` 并传递用户页表

#### userret（用户态）
`userret` : `userret` 利用传入的参数将 `satp` 切换到进程的用户页表。并将 `trapframe` 中保存的用户 `a0` 复制到 `sscratch` 中，为以后与 `TRAPFRAME` 交换做准备。之后 `userret` 从 `trapframe` 中恢复保存的用户寄存器，对 `a0` 和 `sscratch` 做最后的交换，恢复用户 `a0` 并保存 `TRAPFRAME` ，为下一次trap做准备，并使用 `sret` 返回用户空间。

### 4-3 代码阅读：调用系统调用

用户代码将 `exec` 的参数放在寄存器 `a0` 和 `a1` 中，并将系统调用号放在 `a7` 中。 `ecall` 指令进入内核，执行 `uservec` 、 `usertrap` ，然后执行 `syscall` 。

```assembly
# in initcode.S
# exec(init, argv)
.globl start
start:
        la a0, init
        la a1, argv
        li a7, SYS_exec
        ecall
```

`syscall` 从trapframe中的 `a7` 中得到系统调用号，并其作为索引在 `syscalls` 查找，并调用其对应的实现函数。当系统调用函数返回时，返回值记录在 `p->trapframe->a0` 中。系统调用返回负数表示错误，0或正数表示成功。

```c
// syscall.c
void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    // Use num to lookup the system call function for num, call it,
    // and store its return value in p->trapframe->a0
    p->trapframe->a0 = syscalls[num]();
  } else {
  // 如果系统调用号无效， `syscall` 会打印错误并返回-1。
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```

### 4-4 代码阅读：系统调用参数
(寻找参数发生在寄存器已经被存储到trapframe中后)

内核中处理trap的程序可以在trapframe中寻找需要的参数，以下是寻找参数的函数：
```c
static uint64
argraw(int n)
{
  struct proc *p = myproc();
  switch (n) {
  case 0:
    return p->trapframe->a0;
  case 1:
    return p->trapframe->a1;
  case 2:
    return p->trapframe->a2;
  case 3:
    return p->trapframe->a3;
  case 4:
    return p->trapframe->a4;
  case 5:
    return p->trapframe->a5;
  }
  panic("argraw");
  return -1;
}

// Fetch the nth 32-bit system call argument.
void
argint(int n, int *ip)
{
  *ip = argraw(n);
}

// Retrieve an argument as a pointer.
// Doesn't check for legality, since
// copyin/copyout will do that.
void
argaddr(int n, uint64 *ip)
{
  *ip = argraw(n);
}
```

如果参数为指针，内核获取指针后不能使用普通指令从用户提供的地址加载或存储，因为指针可能恶意且内核页表与用户页表映射不一致。一般使用如下的函数来实现安全地将数据复制到用户提供的地址或从用户提供的地址复制数据：

```c
// Fetch the uint64 at addr from the current process.
int
fetchaddr(uint64 addr, uint64 *ip)
{
  struct proc *p = myproc();
  if(addr >= p->sz || addr+sizeof(uint64) > p->sz) // both tests needed, in case of overflow
    return -1;
  if(copyin(p->pagetable, (char *)ip, addr, sizeof(*ip)) != 0)
    return -1;
  return 0;
}

// Fetch the null-terminated string at addr from the current process.
// Returns length of string, not including nul, or -1 for error.
int
fetchstr(uint64 addr, char *buf, int max)
{
  struct proc *p = myproc();
  if(copyinstr(p->pagetable, buf, addr, max) < 0)
    return -1;
  return strlen(buf);
}
```

> 我怎么没感觉非常安全……疑问：检查指针是否安全的部分在？

`copyinstr` 和 `copyout` 实现了安全地从地址读出数据或向地址写入数据。（看了下这个和本章关系不大主要还是内存内容，于是不附这两个函数的代码）

### 4-5 内核空间的trap
在硬件已经被切换到特权模式（内核态）之后， `stvec` 中被写入的值为 `kernelvec` 的地址，而将 `pc` 切换为 `stvec` 中的 `kernelvec` 之后， `kernelvec` 可以将 `satp` 设置为内核页表并将栈指针切换到内核栈。

`kernelvec` 将所有寄存器值保存在原内核线程的堆栈上。之后跳转到 `kerneltrap` 来处理设备中断和异常（内核只能发生这两种可能的trap），如果trap类型为设备中断，调用 `devintr` ;如果trap类型为异常，内核调用 `panic` 并停止执行。

如果由于计时器中断而调用了 `kerneltrap` ，并且进程的内核线程正在运行（而不是调度程序线程）， `kerneltrap` 调用 `yield` 让出CPU，允许其他线程运行。在某个时刻，其中一个线程将退出，并让我们的线程及其 `kerneltrap` 恢复。

### 4-6 缺页故障
xv6对异常的响应方式：

- 用户空间发生异常 -> 直接杀死鼓掌进程

- 内核发生异常 -> panic

实际运用中操作系统会用更丰富的方式来处理异常。例如使用页面故障来处理***写时复制（copy-on-write，cow）fork*** -> 其中的基本设计是父进程和子进程最初共享所有的物理页面，但将它们映射设置为只读。当子进程或父进程执行store指令时，RISC-V CPU会引发一个页面故障异常。作为对这个异常的响应，内核会拷贝一份包含故障地址的页。然后将一个副本的读/写映射在子进程地址空间，另一个副本的读/写映射在父进程地址空间。更新页表后，内核在引起故障的指令处恢复故障处理。因为内核已经更新了相关的PTE，允许写入，所以现在故障指令将正常执行。

***懒分配 (lazy allocation)*** 同样也是使用页面故障实现的内存分配策略：在一个进程调用 `sbrk` 请求增长内存后，内核会为其增长地址空间（但并不分配物理内存），但在页表中令新分配的页全部无效。当新分配的页在被试图读/写时，会发生页面故障，此时内核实际为其分配物理内存。

还有一个策略是***磁盘上分页(paging from disk)*** ， 这个策略就是把懒分配实现到磁盘上。

### 4-7 真实世界
如果将内核内存映射到每个进程的用户页表中（使用适当的PTE权限标志），就不需要特殊的trampoline页了。这也将消除从用户空间trap进入内核时对页表切换的需求。这也可以让内核中的系统调用实现利用当前进程的用户内存被映射的优势，让内核代码直接去间接引用（对地址取值）用户指针。许多操作系统已经使用这些想法来提高效率。

省流：直接把内核内存放一份到用户进程中。

