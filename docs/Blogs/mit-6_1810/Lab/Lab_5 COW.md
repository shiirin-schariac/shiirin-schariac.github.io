第一个写得我想哭的lab。。。。。。

> 目标：实现 copy-on-write 写时复制。

需要修改的函数： `uvmcopy`、 `kalloc.c` 中的一系列函数、`usertrap` 以及 `copyout` 

总体的实现思路：

- 在复制内存时不分配新的内存，而是修改待被复制的内存中的原可读页的标志位将其标识为COW页，然后直接把待被复制的内存映射到目标地址空间中。
- 同时需要实现一个引用标识数组，来标识一个物理地址现在有多少个进程在引用它，只有在引用数为0时才能释放对应的物理内存。
- 修改处理trap的 `usertrap` 使其能够处理 `pagefault` ，当 `pagefault` 是因为尝试写入一个COW页发生时，分配一页的空间并将原COW页的内容复制进去，修改新分配的页为可写入页。
- 修改从内核向用户地址空间写入的 `copyout` 函数，使其在写入前先检查写入页是否为COW页，如是则分配一页的空间并将原COW页的内容复制进去，修改新分配的页为可写入页。

前置步骤：在 `riscv.h` 中添加 `#define PTE_COW (1L << 8)` ，启动 `PTE_COW` 位。

### 修改 `kalloc.c`

接下来根据总体思路修改对应函数，首先在 `kalloc.c` 中增加页面引用部分的内容 :

在 `kmem` 中增加一个数组 `refcount` 用来标记每一个物理地址对应的页被引用的次数，决定这个数组长度的变量在第3章中（主要去看那个内存分布图）：
```c
struct {
  struct spinlock lock;
  struct run *freelist;
  int refcount[(PHYSTOP - KERNBASE)/PGSIZE];
} kmem;
```

增加以下对 `refcount` 进行编辑的函数（注意在 `defs.h` 中添加对应的声明）：

```c
uint
krefer(void* pa) 
{
  return kmem.refcount[(pa - (void*)end) / 4096];
}

void
krefincre(void* pa)
{
  acquire(&kmem.lock);
  kmem.refcount[(pa - (void*)end) / 4096] ++;
  release(&kmem.lock);
}

void
krefdecre(void* pa)
{
  acquire(&kmem.lock);
  kmem.refcount[(pa - (void*)end) / 4096]--;
  release(&kmem.lock);
}
```

同时我们在 `kinit` 中初始化 `refcount` ，因为在 `freearange` 中需要对所有页面调用一次 `kfree` ，所以 `refcount` 需要被初始化填充为1 ：

```c
void
kinit()
{
  initlock(&kmem.lock, "kmem");
  for(int i = 0; i < sizeof(kmem.refcount) / sizeof(kmem.refcount[0]); i ++) 
  {
    kmem.refcount[i] ++;
  }
  freerange(end, (void*)PHYSTOP);
}
```

接着修改 `kfree` ，使首先调用 `krefdecre` 使对应物理页面的被引用数-1，只有当被引用数为0时才释放内存：

```c
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  krefdecre(pa);
  if(!krefer(pa))
  {
    // Fill with junk to catch dangling refs.
    memset(pa, 1, PGSIZE);

    r = (struct run *)pa;

    acquire(&kmem.lock);
    r->next = kmem.freelist;
    kmem.freelist = r;
    release(&kmem.lock);
  }
}
```

最后修改 `kalloc` ，使分配页面时增加被分配物理页面的引用数：

```c
void *
kalloc(void)
{
  struct run *r;

  acquire(&kmem.lock);
  r = kmem.freelist;
  if(r)
    kmem.freelist = r->next;
  release(&kmem.lock);

  if(r)
  {
    krefincre((void*)r);
    memset((char*)r, 5, PGSIZE); // fill with junk
  }
  return (void*)r;
}
```

### 修改 `uvmcopy`

思路就是上面的思路，实现细节见注释：

```c
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;
  // char *mem;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);

    if(flags & PTE_W) { // 原页表项为可写页
      flags = (flags & ~PTE_W) | PTE_COW; // 设置COW位且取消W位
    }
    
    // 直接把原页的物理地址映射到新的地址空间中
    if(mappages(new, i, PGSIZE, pa, flags) != 0) {
      goto err;
    }

    if(flags & PTE_COW) { // 如果原页表项就是COW页
      *pte = PA2PTE(pa) | flags; // 对新的页表项也如此设置
    }

    krefincre((void*)pa); // 原页表项对应的物理地址引用数++
  }
  return 0;

 err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}
```

### 修改 `copyout`

同上：

```c
int
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
  uint64 n, va0, pa0;
  pte_t *pte;

  while(len > 0){
    va0 = PGROUNDDOWN(dstva);
    if(va0 >= MAXVA)
      return -1;
    pte = walk(pagetable, va0, 0);
    if(pte == 0 || (*pte & PTE_V) == 0 || (*pte & PTE_U) == 0 ||
       ((*pte & PTE_W) == 0 && (*pte & PTE_COW) == 0))
    // 写入页要么为可写页要么为COW页
      return -1;
    pa0 = PTE2PA(*pte);

	// 如果该物理地址引用数大于1，对应页表项一定是COW页
	// 也可以用COW位来判断
    if(krefer((void*)pa0) > 1) { 
      char* mem = kalloc();
      if(mem == 0) {
        return -1;
      }
      memmove(mem, (char*)pa0, PGSIZE);

	  // 对新分配的物理页取消COW位，设置W位
      uint flags = (PTE_FLAGS(*pte) & ~PTE_COW) | PTE_W;
      *pte = PA2PTE(mem) | flags;

	  // 使原物理页引用数--
      kfree((void*)pa0);
      pa0 = (uint64)mem;
    }

    n = PGSIZE - (dstva - va0);
    if(n > len)
      n = len;
    memmove((void *)(pa0 + (dstva - va0)), src, n);

    len -= n;
    src += n;
    dstva = va0 + PGSIZE;
  }
  return 0;
}
```

### 修改 `usertrap` 

page fault的 `scause` 为15，所以添加一个处理 `r_scause() == 15` 情况的分支：

```c
// r_stval储存了发生错误时试图访问的虚拟地址
  else if (r_scause() == 15 && r_stval() < MAXVA) {
    uint64 addr = r_stval();
    // 使用walk找到该虚拟地址对应的最后一级页表项地址
    pte_t* pte = walk(p->pagetable, addr, 0);
    // 判断这个页表项是否是COW页
    // 如不是，说明是非法的缺页故障，直接杀死进程
    if ((*pte & PTE_COW) == 0) {
      setkilled(p);
      exit(-1);
    }
    uint64 pa = PTE2PA(*pte);

	// 分配物理页
    char* mem = kalloc();
    
    if (mem == 0) {
      setkilled(p);
      exit(-1);
    }
    memmove(mem, (char* )pa, PGSIZE);
    kfree((void*)pa);

	// 对新分配的物理页设置标志位
    uint flags = PTE_FLAGS(*pte) | PTE_W;
    *pte = PA2PTE(mem);
    *pte |= flags;
  }
```


大概就是这样，写的时候 `cowtest` 倒是一次过了，但是跑 `usertests -q` 的时候遇到两个问题：一个是访问的虚拟地址必须不能超过 `MAXVA` ，需要加一个针对这个的判断；还有一个是处理缺页故障时必须要判断是否是 COW 的报错。。这两个差点没查死我，特以警醒后人。。。