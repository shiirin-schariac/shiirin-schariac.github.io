## Speed up system calls
> 要求我们在每个进程被创建时都创建一页只读映射到虚拟地址USYSCALL（该地址已被定义在memlayout.h），在这一页中存储一个 `usyscall` 结构体，其中存储了该进程的pid，这样每次想要获取该进程的pid时就不用调用系统调用了

实现思路：将 `usyscall` 结构体添加到 `proc` 结构体中。初始化进程时同时初始化 `usyscall` 结构体，把pid放进去。最后只要把 `usyscall` 结构体的物理地址作为参数传给 `mappages` ，将其映射到虚拟地址USYSCALL就可以。注意需要补充释放内存函数中关于 `usyscall` 结构体的部分。

`proc.c`
```c
//  in allocproc
  if((p->usyscall = (struct usyscall *)kalloc()) == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  p->usyscall->pid = p->pid;
```

```c
//	in freeproc
  if(p->usyscall)
    kfree((void*)p->usyscall);
  p->usyscall = 0;
```

```c
//  in proc_pagetable
  if(mappages(pagetable, USYSCALL, PGSIZE, (uint64)p->usyscall, PTE_R | PTE_U) < 0) {
  // 设置权限为只读且用户程序可读
    uvmunmap(pagetable, TRAPFRAME, 1, 0);
    uvmunmap(pagetable, TRAMPOLINE, 1, 0);
    uvmfree(pagetable, 0);
    return 0;
  }
```

```c
//  in proc_freepagetable
  uvmunmap(pagetable, USYSCALL, 1, 0);
```

`proc.h`
```c
//  in struct proc
  struct usyscall *usyscall;
```


## Print a page table
> 要求我们打印pid为1的进程的多级页表

实现思路：为了实现分级打印，我们先实现一个 `pteprint` 函数（其实是因为题目要求 `vmprint` 只能有一个页表指针参数……）。如果该项有效，根据级别打印前缀，然后打印地址，如果该级页表项中的地址指向更低一级的页表，则调用递归，且级别++。

```c
void 
pteprint(pagetable_t pagetable, int level) 
{
  for (int i = 0; i < 512; i++)
  {
    pte_t pte = pagetable[i];

    if (pte & PTE_V)
    {
      // this PTE points to a lower-level page table.
      uint64 child = PTE2PA(pte);
      for(int i = level; i >= 0; i--) {
        printf(" ..");
      }
      printf("%d: pte %p pa %p\n", i, pte, child);
      if ((pte & (PTE_R | PTE_W | PTE_X)) == 0)
        pteprint((pagetable_t)child, level + 1);
    }
  }
}

void
vmprint(pagetable_t pagetable) 
{
  printf("page table %p\n", pagetable);
  pteprint(pagetable, 0);
}
```

## Detect which pages have been accessed

> 要求我们实现一个名为 `pgaccess` 的系统调用。传入一个起始地址va，要查询的页数pgs以及一个位掩码abits， 检查从va开始的pgs页是否被访问过，结果按位存储在abits中。

实现思路：从va起始每次递增 `PGSIZE` （递增一页），使用 `walk` 寻找该虚拟地址对应的页表项，查询其 `PTE_A` ，如果为有效值，则在位掩码中设置并清空 `PTE_A` （否则 `PTE_A` 将为一直有效）。

关于位掩码如何传输到用户空间中：用户实际上传入一个位掩码起始的地址，我们在系统调用内部使用一个 `unsigned int` 类型的位掩码记录数据后，用 `copyout` 函数将这个位掩码拷贝到用户传入的地址中。

（需要注意关于 `PTE_A` 的设置，由RISCV硬件自动完成了= =，不需要自己再在访问页表的函数里设置）

先在 `riscv.h` 中添加 `PTE_A` 的定义：
```c
#define PTE_A (1L << 6) // if the page is accessed
```

在 `syspro.c` 中进行如下实现：

```c
#ifdef LAB_PGTBL
int
sys_pgaccess(void)
{
  // lab pgtbl: your code here.
  uint64 va;
  uint64 mask;
  int pgs;
  unsigned int abits = 0;
  pagetable_t pagetable = myproc()->pagetable;

  argaddr(0, &va);

  argint(1, &pgs);
  if(pgs > 32)
    return -1;

  argaddr(2, &mask);
  
  for(int i = 0; i < pgs; i++)
  {
    pte_t *pte = walk(pagetable, va + PGSIZE * i, 0);
    if (pte == 0)
      return -1;

    if (*pte & PTE_A)
    {
      abits |= 1 << i; // 设置位掩码的对应位
      *pte &= ~PTE_A; // 清空PTE_A位
    }
  }

  if (copyout(pagetable, mask, (char *)&abits, sizeof(abits)) < 0)
  // 将位掩码拷贝到用户传入的地址中
  {
    return -1;
  }

  return 0;
}
#endif
```