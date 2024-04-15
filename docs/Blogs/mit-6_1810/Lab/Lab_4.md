### backtrace
> 要求我们在 `kernel/printf.c` 里添加一个 `backtrace` 函数，实现调用它时能打印栈里几次调用的返回地址。实现完后在 `sys_sleep` 系统调用里增加一个对 `printf` 的调用，打印在调用 `sys_sleep` 时栈里的返回地址。

在 `def.h` 中添加 `backtrace` 的原型后，在 `kernel/printf.c` 中添加如下代码：
```c
void
backtrace(void)
{
  uint64 fp = r_fp(); // get the frame pointer of the process

  printf("backtrace:\n");
  while(fp != PGROUNDDOWN(fp)) { // check if the fp is at the top of the stack
    printf("%p\n", *(uint64 *)(fp - 0x8)); // the offset to return address is 8
    fp = *(uint64 *)(fp - 0x10); // the offset to next fp is 16
  }
}
```

在 `sys_sleep` 系统调用中调用 `backtrace` 后，使用 `bttest` 指令测试，输出如下:

```shell
$ bttest
backtrace:
0x0000000080002272
0x00000000800020e0
0x0000000080001dd6
```

在另一个终端中输入 `riscv64-unknown-elf-addr2line -e kernel/kernel` 后，逐个输入上面的地址，得到：

```shell
riscv64-unknown-elf-addr2line -e kernel/kernel
0x0000000080002272
/xv6-labs-2023/kernel/sysproc.c:72
0x00000000800020e0
/xv6-labs-2023/kernel/syscall.c:141 (discriminator 1)
0x0000000080001dd6
/xv6-labs-2023/kernel/trap.c:76
```

返回的行数为trap结束后的返回地址。

### alarm
> 要求我们实现一个系统调用 `sigalarm(interval, handler)` ，在调用该系统调用后，每间隔 `interval` 个CPU时间后，会调用 `handler` 来处理。

test0不要求我们具体实现 `sigreturn` （直接返回0就可以了），先讲讲如何实现test0：

需要使用到的变量的定义：

- 在进程中定义一个变量 `ticks` 表示（上次调用结束后剩下的）CPU时间，并用这个变量来记录调用后一共经过了几个CPU时间。
- 在进程中定义一个变量 `ticks_interval` 来记录传入的 `interval` 参数。
- 在进程中定义一个变量 `fnret` 来记录传入的处理函数 `handler` 的地址。

函数部分实现思路：

在 `sysproc` 里添加一个系统调用 `sys_sigalarm` ：

```c
uint64
sys_sigalarm(void)
{
  int ticks;
  uint64 fnret;

  argint(0, &ticks);
  argaddr(0, &fnret);

  struct proc* p = myproc();
  acquire(&p->lock);
  p->ticks_interval = ticks;
  p->fnret = fnret;
  p->ticks = 0;
  release(&p->lock);

  return 0;
}
```

因为每经过一个CPU时间CPU会进行一次定时器中断，这个定时器中断会由 `trap.c` 中的 `usertrap` 调用 `devintr` 处理， `devintr` 在该中断为定时器中断时返回值为2，所以我们在 `which_dev` 为2时使 `p->ticks` 加1，并判断当前情况下是否需要调用处理函数 `handler` ，如果需要，则将该中断的**返回地址**设为处理函数 `handler` 的地址，这样在调用 `usertrapret` 的时候返回的位置就是处理函数 `handler` ：

```c
void
usertrap(void)
{
  // ...
  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2)
  {
    acquire(&p->lock);
    p->ticks ++;
    if(p->ticks_interval != 0 && p->fnret && p->ticks % p->ticks_interval == 0)
    {
      p->ticks = 0;
      p->trapframe->epc = p->fnret;
    }
    release(&p->lock);
    
    yield();
  }
}
```

（本解析中忽视了函数声明、添加系统调用等部分）
测试test0，通过。

考虑一般的定时器中断，内核处理中断后调用 `usertrapret` 返回中断发生处，但是经过上面的修改， `usertrapret` 返回的是处理函数 `handler` 的位置。为了正确地返回中断发生处，我们需要在处理函数中调用 `sigreturn` ，然后在 `sigreturn` 中返回原本的中断发生处。

为了实现这点，我们需要在进程中添加一个 `trapframeret` 用于储存原来的寄存器与返回地址（记得要对 `trapframeret` 进行初始分配内存与释放内存的设置）：

```c
// in usertrap
if(p->alarm_interval != 0 && p->fnret && p->ticks % p->alarm_interval == 0)
{
    memmove(p->trapframeret, p->trapframe, sizeof(struct trapframe));
    p->ticks = 0;
    p->trapframe->epc = p->fnret;
}
```

实现 `sigreturn` ：

```c
uint64 
sys_sigreturn(void)
{
  struct proc* p = myproc();
  acquire(&p->lock);
  memmove(p->trapframe, p->trapframeret, sizeof(struct trapframe));
  release(&p->lock);

  return 0;
}
```

这样test1就能通过了，下面是test2和test3的要求（好麻烦啊这俩并一起写了）：

- test2: 要求处理函数 `handler` 未返回前不能进行新的处理函数调用，实现思路就是在 `usertrap` 的判断处添加一个变量 `handlerret` 判断现在处理函数是否已返回就可以了。
- test3: 因为 `sigreturn` 是系统调用，所以要把寄存器 `a0` 中的值作为返回值返回。

最终的 `sigreturn` :

```c
uint64 
sys_sigreturn(void)
{
  struct proc* p = myproc();

  acquire(&p->lock);
  p->handlerret = 1;
  memmove(p->trapframe, p->trapframeret, sizeof(struct trapframe));
  release(&p->lock);

  return p->trapframe->a0;
}
```

最终的 `usertrap` :

```c
void
usertrap(void)
{
	//...

	if(which_dev == 2)
  {
    acquire(&p->lock);
    p->ticks ++;
    if(p->alarm_interval && p->handlerret && p->ticks % p->alarm_interval == 0)
    {
      memmove(p->trapframeret, p->trapframe, sizeof(struct trapframe));
      p->handlerret = 0;
      p->trapframe->epc = p->fnret;
    }
    release(&p->lock);

    yield();
  }

  usertrapret();
}
```