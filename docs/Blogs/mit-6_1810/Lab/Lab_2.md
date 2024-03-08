macOS使用gdb调试xv6的过程（前提：已经安装并下载gdb，安装步骤可参考[这个链接](https://oslab.kaist.ac.kr/xv6-install/?ckattempt=3)）：
```shell
$ cd xv6-labs-2023
$ make qemu-gdb
```

重新开一个窗口
```shell
$ cd xv6-labs-2023
$ i386-elf-gdb –version
```

现在是在gdb模式下运行xv6
没有试过6.8以上的gdb，这个版本实在太老了，但是先这样吧
#### trace
先按照hint在各种文件中添加存根等，在 `proc` 下增加变量 `mask` 用于记录需要追踪的系统调用的编号，同时修改 `fork` 函数使子进程继承父进程的 `mask` 变量
在文件 `sysproc.c` 下添加函数：
```c
uint64
sys_trace(int)
{
	int mask;
	
	argint(0, &mask); // gather the argument from user space
	if(mask < 0) {
		mask = 0;
	}
	myproc()->trace_mask = mask; // assign the argument to the proc srtuct

	return 0;
}
```

最后在 `syscall.c` 文件中编辑 `syscall` 函数：
```c
// if the syscall is being traced
if((p->trace_mask >> num) & 1) {
	printf("%d: %s -> %d\n", p->pid, syscall_name[num], p->trapframe->a0);
}
```

#### sysinfo
首先按照同样的步骤添加存根等
在 `kalloc.c` 中增加一个统计空余内存的函数：
```c
uint64
countfree(void)
{
	struct run *r;
	uint64 sumfree = 0;

	for(r = kmem.freelist; r; r = r->next) {
		sumfree += 4096;
	}

	return sumfree;
}
```

在 `proc.c` 中增加一个统计状态非UNUSED的进程：
```c
// count how many process is not UNUSED
uint64
countproc(void)
{
	uint64 procsum = 0;
	struct proc *p;
	
	for (p = proc; p < &proc[NPROC]; p++)
	{
		if (p->state != UNUSED)
		{
			procsum ++;
		}
	}
	
	return procsum;
}
```

在文件 `sysproc.c` 下添加函数：
```c
uint64
sys_sysinfo(void) 
{
  uint64 si;
  struct sysinfo info;

  argaddr(0, &si);
  if(!si) {
    return -1;
  }

  info.freemem = countfree();
  info.nproc = countproc();

  if (copyout(myproc()->pagetable, si, (char *)&info, sizeof(info)) < 0)
  {
    return -1;
  }
  return 0;
}
```