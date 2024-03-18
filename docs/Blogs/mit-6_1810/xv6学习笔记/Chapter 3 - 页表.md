每个进程都会被分到一块独立的内存空间，这块内存空间中的地址被映射到一张或几张页表上，进程使用页表访问内存。页表机制保证了进程不会访问非法的内存（即隔离了不同进程的内存空间），同时为进程提供了一个在它看来“从0开始”的内存空间，方便其访问。

xv6同时利用这个机制实现了一些技巧：在几个地址空间中映射同一内存（ trampoline 页），以及用一个未映射页来保护内核栈和用户栈。

阅读前需要分辨的基础概念：

- 页与页表的关系：页为一块物理内存，页表中储存了一系列的页表项，每个页表项指向一页即一块物理内存。假设一页为4096个字节，虚拟地址a为一页的起始地址，则a ～ a+4095指向一个页表项，a+4096指向下一个页表项。
- 虚拟地址 = 一个查找页表项的地址 + 偏移量。物理地址 = 页表项中储存的物理内存块地址 + 偏移量。
### 3-1 页表硬件
> 一个RISC-V页表在逻辑上是一个由2²⁷（134,217,728）个<b>页表项（Page Table Entry, PTE）</b>组成的数组。每个**PTE**包含一个44位的<b>物理页号（Physical Page Number, PPN）</b>和一些标志位。分页硬件通过利用39位中的高27位索引到页表中找到一个**PTE**来转换一个虚拟地址，并计算出一个56位的物理地址，它的前44位来自于**PTE**中的**PPN**，而它的后12位则是从原来的虚拟地址复制过来的。

页表硬件负责将页表中的虚拟地址映射为物理地址。虚拟地址为页表标号( index ) + 偏移量( offset )。页表标号对应了页表中的一项，也即页表项的地址，此页表项为物理内存中的起始位置：该起始位置加上偏移量即为虚拟地址对应的物理地址。

即页表可以为简单地被看为一个存储了多个物理内存起始位置的数组，结合虚拟地址中的偏移量，虚拟地址可以被转换为物理地址。

每个 PTE 都包含标志位，用于告诉分页硬件相关的虚拟地址被允许怎样使用。`PTE_V` 表示 PTE 是否存在：如果没有设置，对该页的引用会引起异常（即不允许）。`PTE_R` 控制是否允许指令读取该页。`PTE_W` 控制是否允许指令向该页写入。`PTE_X` 控制 CPU 是否可以将页面的内容解释为指令并执行。`PTE_U` 控制是否允许用户态下的指令访问页面；如果不设置 `PTE_U`， 对应 PTE 只能在内核态下使用。

其中X、W、R三位为leaf PTE（不可修改的），riscv文档中对于每一页持有的权限位代表的该页性质的说明：

| X   | W   | R   | Meaning              |
| --- | --- | --- | -------------------- |
| 0   | 0   | 0   | 指向下一级页表的指针 |
| 0   | 0   | 1   | 只读页               |
| 0   | 1   | 0   | 保留以等待将来使用   |
| 0   | 1   | 1   | 可读可写页           |
| 1   | 0   | 0   | 只执行页             |
| 1   | 0   | 1   | 可读可执行页         |
| 1   | 1   | 0   | 保留以等待将来使用   |
| 1   | 1   | 1   | 可读可写可执行页     |

> 关于non-leaf PTE: For non-leaf PTEs, the D, A, and U bits are reserved for future use and must be cleared by software for forward compatibility.


> 对于lab3中最后一个任务需要用到的 `PTE_A` ，文档中进行的说明如下：Each leaf PTE contains an accessed (A) and dirty (D) bit. The A bit indicates the virtual page has been read, written, or fetched from since the last time the A bit was cleared. The D bit indicates the virtual page has been written since the last time the D bit was cleared.
> 
> 本lab中我们先考虑 `PTE_A` 是如何实现的：对于一个 `PTE_A` 为0的页表，一旦被访问，就设置其 `PTE_A` 为1。 `PTE_A` 永远不应该被主动清除（为了提高效率），在本lab中，我们在调用过 `pgaccess` 之后清除 `PTE_A` 以判断其是否在上一次调用 `pgaccess` 后被访问。



CPU内有 `satp` 寄存器记录需要使用的页表的物理位置，不同的CPU可以记录不同的页表，保证每个进程都有自己的页表所描述的内存空间。

通过已知的页表物理位置，结合虚拟地址中的页表项标号，可以递进式地在xv6的三级页表中找到对应的物理内存块的地址。（这一步主要由硬件完成）

### 3-2 内核地址空间

xv6为每个进程维护一个映射其用户地址空间的页表。对内核，维护一个单独的描述内核地址空间的页表，这个页表的配置是固定的，以保证内核可以通过虚拟地址访问物理内存。

内核中的虚拟地址映射从地址 `0x80000000` 开始，到（至少） `0x86400000` 结束。 `0x80000000` 以下的内存用于I/O设备，内核可以通过读取/写入这些特殊的物理地址与设备进行交互。内核对设备寄存器和RAM（物理内存）使用直接映射。

内核中内核栈页与trampoline页不是直接映射。

- trampoline页：一个trampoline物理页（存放trampoline代码）被映射两次，一次在虚拟地址空间顶部，一次直接映射。同时它在用户的地址空间中被多次映射。

- 内核栈页：内核栈页被映射到地址空间的顶端后，在其后放置一个 `PTE_V` 无效的页（守护页），以保证在内核栈溢出时报错。

### 3-3 代码阅读：创建地址空间
本节中的主要代码位于 `kernel/vm.c` 。

核心数据结构： `pagetable_t` ，本质为一个指向根页表页的指针，它可以**是**内核页表也可以**是**进程页表。

首先 `main` 函数调用 `kvminit` 来创建内核页表并分配一页物理内存来存放根页表页，然后调用 `kvmmap` 将内核所需要的硬件资源映射到物理地址。这些资源包括内核的指令和数据。

产生映射时的核心函数：
- `mappages` ：它对每个虚拟地址先调用 walk 查找其对应的最后一级页表项的地址，然后初始化页表项，使其持有相关的物理页号、所需的权限以及`PTE_V`来标记 PTE 为有效。
- `walk` ：在页表中查找并返回给定虚拟地址对应的最后一级页表项的地址。如果查找到的页表项无效，那么所需的物理页还没有被分配 -> 此时若 `alloc` 参数被设置，`walk` 会分配一个新的页表页，并把它的物理地址放在 PTE 中。

```c
// va=虚拟内存起始地址，pa=物理内存起始地址，size=待映射地址空间大小。
int
mappages(pagetable_t pagetable, uint64 va, uint64 size, uint64 pa, int perm)
// Create PTEs for virtual addresses starting at va that refer to physical addresses starting at pa.
{
  uint64 a, last;
  pte_t *pte;

  // 以页为单位，所以va和size都必须匹配PGSIZE
  if((va % PGSIZE) != 0)
    panic("mappages: va not aligned");

  if((size % PGSIZE) != 0)
    panic("mappages: size not aligned");

  if(size == 0)
    panic("mappages: size");

  a = va; // 当前虚拟地址
  last = va + size - PGSIZE; // 虚拟地址范围的结束地址
  for(;;){
    if((pte = walk(pagetable, a, 1)) == 0) // 调用 `walk` 函数来查找虚拟地址 `a` 对应的页表项的地址，并将结果存储在变量 `pte` 中。如果查找失败（即返回值为0），则说明页表项不存在，函数返回-1。
      return -1;
    if(*pte & PTE_V) // 检查页表项是否已经被映射。
      panic("mappages: remap");
    *pte = PA2PTE(pa) | perm | PTE_V; // 将页表项设置为有效，并存储相应的物理页号、权限标志等信息。`PA2PTE(pa)` 是将物理地址转换为页表项格式的宏，`perm` 表示要设置的权限标志，`PTE_V` 表示页表项有效。
    if(a == last)
      break;
    // 将地址向后移动一页大小，以处理下一个虚拟地址。
    a += PGSIZE;
    pa += PGSIZE;
  }
  return 0;
}
```

```c
pte_t *
walk(pagetable_t pagetable, uint64 va, int alloc)
{
  if(va >= MAXVA) // 检查虚拟地址是否超出了最大虚拟地址范围。如果超出了最大虚拟地址范围
    panic("walk");

  for(int level = 2; level > 0; level--) {
    pte_t *pte = &pagetable[PX(level, va)];
    // 根据当前级别和虚拟地址计算出对应的页表项的地址，并将其存储在变量 `pte` 中。
    
    if(*pte & PTE_V) {
	// 检查页表项是否有效。有效说明该页表项中的地址有效。
      pagetable = (pagetable_t)PTE2PA(*pte);
      // 将该页表项中的地址转换为物理地址储存在pagetable中
      // 若该页表项并非最后一级，则使用该地址寻找下一级页表
      // 若该页表项为最后一级，则返回该页表项中存储的物理地址
    } else {
      if(!alloc || (pagetable = (pde_t*)kalloc()) == 0)
      // 若alloc为1，使用kalloc分配新的物理页
        return 0; // 分配不成功或不分配
      memset(pagetable, 0, PGSIZE); // 初始化新分配的物理页
      *pte = PA2PTE(pagetable) | PTE_V; // 设置相应的权限标志，并将下一级页表的物理地址存储在当前页表项中。
    }
  }
  return &pagetable[PX(0, va)]; // 返回最低级别的页表项的地址，即对应给定虚拟地址的页表项的地址。
}
```

> 疑问：怎么感觉实际上没有看到分配物理内存的部分？
> 回答：本过程不涉及物理空间的分配，是在已经分配好的物理空间上初始化地址空间，物理空间的分配见kalloc函数。

内核页表被映射后，根页表页的物理地址会被写入CPU的 `satp` 寄存器中，之后CPU会使用这个页表进行映射。

此处需要回顾计组中的TLB知识，即页表中的快表。xv6改变页表后，会使该页表在TLB中的对应索引项失效，否则可能导致进程访问非法的内存空间。

### 3-4 物理内存分配
xv6 使用内核地址结束到 `PHYSTOP` （物理地址结束的位置）之间的物理内存来进行运行时分配。它每次分配和释放整个4096 字节的页面。它通过保存空闲页链表，来记录哪些页是空闲的。分配包括从链表中删除一页；释放包括将释放的页面添加到空闲页链表中。

### 3-5 代码阅读：物理内存分配器
本节中的主要代码位于 `kernel/kalloc.c` 。

调用 `kalloc` 时，分配器从一个存储空闲页表的链表 `freelist` 中查找空闲的页表并返回一个内核可使用的指向这个页表的指针（=将其分配给对应进程），并将其从 `freelist` 中删除。

源代码如下，根据文档进行了注释补充：
```c
// Physical memory allocator, for user processes,
// kernel stacks, page-table pages,
// and pipe buffers. Allocates whole 4096-byte pages.

// omit the library part

void freerange(void *pa_start, void *pa_end);

extern char end[]; // first address after kernel.
                   // defined by kernel.ld.

// 单个指向空闲页表的指针
struct run {
  struct run *next;
};

// 一个抽象的内存 -> freelist是一个储存空闲页表的链表
struct {
  struct spinlock lock;
  struct run *freelist;
} kmem;

// 首先调用kinit初始化kmem
// 并释放从内核内存结束位置到物理内存结束位置的所有页表
void
kinit()
{
  initlock(&kmem.lock, "kmem");
  freerange(end, (void*)PHYSTOP);
}


// 将给出的物理地址始末范围内的页表全部设为空闲页表
// = 释放指定范围内的页表内存
void
freerange(void *pa_start, void *pa_end)
{
  char *p;
  p = (char*)PGROUNDUP((uint64)pa_start);
  for(; p + PGSIZE <= (char*)pa_end; p += PGSIZE)
    kfree(p);
}

// Free the page of physical memory pointed at by pa,
// which normally should have been returned by a
// call to kalloc().  (The exception is when
// initializing the allocator; see kinit above.)
// 用于释放页表内存，释放后将空页表加入freelist
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  // 填充垃圾使对该页表的访问无效
  memset(pa, 1, PGSIZE);

  // 指向被释放的页表
  r = (struct run*)pa;

  acquire(&kmem.lock);
  // 将r加入freelist的头位置
  r->next = kmem.freelist;
  kmem.freelist = r;
  release(&kmem.lock);
}

// Allocate one 4096-byte page of physical memory.
// Returns a pointer that the kernel can use.
// Returns 0 if the memory cannot be allocated.
void *
kalloc(void)
{
  struct run *r; 

  acquire(&kmem.lock);
  r = kmem.freelist; // 从freelist的头获取一个空闲页表
  if(r)
    kmem.freelist = r->next;
  release(&kmem.lock);

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}

```

### 3-6 进程地址空间
一个进程要求更多物理空间的步骤：xv6使用 `kalloc` 分配物理页并将指向该物理页的PTE添加到进程页表中，设置其PTE项。

需要注意的是内核会映射一页trampoline页到每一个用户地址空间的顶端（用于跳转至内核态），所以trampoline页的物理内存页在所有地址空间中都被映射了一遍。

在用户地址空间中，用户栈的下方会存在一页守护页来防止进程访问用户栈溢出而需要使用的非法内存。

### 3-7 代码阅读：sbrk
`sbrk` 是一个进程收缩或增长内存的系统调用。该系统调用由函数 `growproc` 实现:

```c
// Grow or shrink user memory by n bytes.
// Return 0 on success, -1 on failure.
int
growproc(int n)
// n为正数增长内存，为负数释放内存
{
  uint64 sz;
  struct proc *p = myproc();

  sz = p->sz;
  if(n > 0){
    if((sz = uvmalloc(p->pagetable, sz, sz + n, PTE_W)) == 0) 
    // 使用uvmalloc通过kalloc分配物理内存
    // 并使用mappages将页表项添加到用户页表中
    {
      return -1;
    }
  } else if(n < 0)
  {
    sz = uvmdealloc(p->pagetable, sz, sz + n);
    // uvmdealloc通过uvmunmap使用walk来查找页表项
    // 查找到对应页表项后调用kfree来释放其引用的物理内存
  }
  p->sz = sz;
  return 0;
}
```

该函数中使用的 `uvmalloc` :
```c
// Allocate PTEs and physical memory to grow process from oldsz to
// newsz, which need not be page aligned.  Returns new size or 0 on error.
uint64
uvmalloc(pagetable_t pagetable, uint64 oldsz, uint64 newsz, int xperm)
{
  char *mem;
  uint64 a;

  if(newsz < oldsz)
    return oldsz;

  oldsz = PGROUNDUP(oldsz);
  for(a = oldsz; a < newsz; a += PGSIZE){
    mem = kalloc(); // mem为指向空闲页的指针
    if(mem == 0){
      uvmdealloc(pagetable, a, oldsz);
      return 0;
    }
    memset(mem, 0, PGSIZE);
    if(mappages(pagetable, a, PGSIZE, (uint64)mem, PTE_R|PTE_U|xperm) != 0)
    // 使用mappages对新分配的页进行映射，将其加入到用户页表中
    {
      kfree(mem); // 如果mappages失败，释放新分配的内存
      uvmdealloc(pagetable, a, oldsz);
      return 0;
    }
  }
  return newsz;
}
```

该函数中使用的 `uvmdealloc` 及 `uvmunmap` :
```c
// Deallocate user pages to bring the process size from oldsz to
// newsz.  oldsz and newsz need not be page-aligned, nor does newsz
// need to be less than oldsz.  oldsz can be larger than the actual
// process size.  Returns the new process size.
uint64
uvmdealloc(pagetable_t pagetable, uint64 oldsz, uint64 newsz)
{
  if(newsz >= oldsz)
    return oldsz;

  if(PGROUNDUP(newsz) < PGROUNDUP(oldsz)){
    int npages = (PGROUNDUP(oldsz) - PGROUNDUP(newsz)) / PGSIZE;
    // 计算需要移除的页数
    uvmunmap(pagetable, PGROUNDUP(newsz), npages, 1);
  }

  return newsz;
}
```

```c
// Remove npages of mappings starting from va. va must be
// page-aligned. The mappings must exist.
// Optionally free the physical memory. -> ????
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  uint64 a;
  pte_t *pte;

  if((va % PGSIZE) != 0)
    panic("uvmunmap: not aligned");

  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
	  // 使用walk对每一页起始的虚拟地址查找其对应的页表项
	  // 该页表项对应一页，所以虚拟地址以PGSIZE递增
    if((pte = walk(pagetable, a, 0)) == 0) 
      panic("uvmunmap: walk");
    if((*pte & PTE_V) == 0)
      panic("uvmunmap: not mapped");
    if(PTE_FLAGS(*pte) == PTE_V)
      panic("uvmunmap: not a leaf");
    if(do_free){
      uint64 pa = PTE2PA(*pte);
      kfree((void*)pa);
    }
    *pte = 0;
  }
}
```

> 疑问：没有看到 `uvmunmap` 可选使物理内存无效的部分？

### 3-8 代码阅读：exec
`exec` 是创建一个用户地址空间的系统调用。它读取储存在文件系统上的文件用来**初始化**一个用户地址空间。其将对应的文件中需要载入内存的部分载入用户地址空间的步骤为：

打开二进制文件路径 -> 读取ELF头，检查该文件是否是ELF类型 -> 使用`proc_pagetable` 分配一个没有使用的页表 -> 使用 `uvmalloc` 为每一个 ELF 段分配内存 -> `loadseg` 使用 `walkaddr` 找到被分配的内存的物理地址，在该地址写入 ELF 段的每一页，页的内容通过 `readi` 从文件中读取。

`exec` 本质为创造一个新的内存映象载入指定二进制文件来执行系统调用，如果系统调用执行成功，旧的映像（即调用 `exec` 的进程）就会被释放。

### 思考部分
对内存映象的产生过程做一个总结

初始化进程内存（使用 `exec` 或单纯初始化进程时分配trapframe页与trampoline页？） -> 初始化完成后，进程可以调用 `sbrk` 增加或减少内存 -> 以增加内存为例： `uvmalloc` 使用 `kalloc` 分配物理内存后，使用 `mappages` 将物理地址放入虚拟地址对应的页表项中（该页表项的地址由 `walk` 查出）

推荐阅读：[RISC-V privileged architecture manual](https://github.com/riscv/riscv-isa-manual/releases/download/Ratified-IMFDQC-and-Priv-v1.11/riscv-privileged-20190608.pdf)
