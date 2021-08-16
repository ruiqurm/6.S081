这个文件记录阅读相关函数做的笔记  
本lab中，是[`kernel/memlayout.h`](kernel/memlayout.h), [`kernel/vm.c`](kernel/vm.c)和[`kernel/kalloc.c`](kernel/kalloc.c)


# `vm.c`
首先看`vm.c`。这里面核心的结构体是` pagetable_t`，核心的函数是`walk`和`mappages`。  

## 通用
### `mappages`
给定虚拟地址和内存大小，分配一段物理地址。
```c
int
mappages(pagetable_t pagetable, uint64 va, uint64 size, uint64 pa, int perm)
{
  uint64 a, last;
  pte_t *pte;

  a = PGROUNDDOWN(va); //计算对齐后的虚拟地址
  last = PGROUNDDOWN(va + size - 1); //计算最后一个需要分配的页的虚拟地址
  for(;;){
    if((pte = walk(pagetable, a, 1)) == 0)
      return -1;
    if(*pte & PTE_V)
      panic("remap");
    *pte = PA2PTE(pa) | perm | PTE_V; // 把物理地址转换成PTE的
	//PA2PTE 先右移12位，再左移10位；左移的10位标记位
	// perm是标记位
    if(a == last)
      break;
    a += PGSIZE;
    pa += PGSIZE;
  }
  return 0;
}
```
### `walk`
返回虚拟地址对应的PTE项  
如果`alloc`不等于0在找不到页表的时候自动创建  
```c
pte_t *
walk(pagetable_t pagetable, uint64 va, int alloc)
{
  if(va >= MAXVA) // 大于最大虚拟内存空间 MAXVA=2^38 留了1bit为了防止符号拓展问题
    panic("walk");

  for(int level = 2; level > 0; level--) {
    pte_t *pte = &pagetable[PX(level, va)]; // PX是取对应level的index
    if(*pte & PTE_V) { // 如果有效
      pagetable = (pagetable_t)PTE2PA(*pte);  // 合成新的pagetable地址
    } else {
      if(!alloc || (pagetable = (pde_t*)kalloc()) == 0) //不允许分配 或者 无法分配新的页表
        return 0;
      memset(pagetable, 0, PGSIZE);
      *pte = PA2PTE(pagetable) | PTE_V; // 创建新的页表
    }
  }
  return &pagetable[PX(0, va)]; // 返回真实的PTE
}
```
### `walkaddr`
对walk的封装，只能用来查询用户的页表

### `copyout`
从内核空间到用户空间复制`len`个字节。  
```c
int
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
  uint64 n, va0, pa0;

  while(len > 0){
    va0 = PGROUNDDOWN(dstva);
    pa0 = walkaddr(pagetable, va0);
    if(pa0 == 0)
      return -1;
    n = PGSIZE - (dstva - va0);
    if(n > len)
      n = len;
    memmove((void *)(pa0 + (dstva - va0)), src, n); // 移动物理内存内容 src->dst

    len -= n;
    src += n;
    dstva = va0 + PGSIZE;
  }
  return 0;
}
```
### `copyin`
从用户空间到内核空间拷贝len字节  
和`copyout`类似
### `copyinstr`
从用户空间拷贝一个字符串，字符串以`\0`结尾。


## kernel相关

### `kvminit`
创建一个内核虚拟空间。   
使用`kalloc`创建一个页表。  
分配了`UART0`、`VIRTIO0`、`CLINT`、`KERNBASE`、`etext`和`TRAMPOLINE`。  
其中`TRAMPOLINE`分配在虚拟地址最顶上一页。
![](https://raw.githubusercontent.com/ruiqurm/image_host/master/blog/20210814165126.png)

### `kvmmap`
创建一个内核的映射，使用的是`mappages`函数


### `kvmpa`
翻译内核虚拟地址到物理地址；只在访问内核栈上需要使用

### `kvminithart`

## user空间相关
### `uvmunmap` 
移除连续n页的页表映射，可以选择是否要同样释放物理内存(`do_free`)
```c
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  uint64 a;
  pte_t *pte;

  if((va % PGSIZE) != 0)
    panic("uvmunmap: not aligned");

  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0)
      panic("uvmunmap: walk");
    if((*pte & PTE_V) == 0)
      panic("uvmunmap: not mapped");
    if(PTE_FLAGS(*pte) == PTE_V) //似乎是叶子节点的valid是false
      panic("uvmunmap: not a leaf");
    if(do_free){
      uint64 pa = PTE2PA(*pte);
      kfree((void*)pa);
    }
    *pte = 0;
  }
}
```
### `uvmcreate`  
创建一个全为0的用户页表  

### `uvminit`
把用户空间其实代码加载到地址0.  
首先会分配一个物理内存，然后把页表虚拟地址0映射到这个内存上。  
最后拷贝src指向的代码到物理内存。
```c
void
uvminit(pagetable_t pagetable, uchar *src, uint sz)
{
  char *mem;

  if(sz >= PGSIZE)
    panic("inituvm: more than a page");
  mem = kalloc();
  memset(mem, 0, PGSIZE);
  mappages(pagetable, 0, PGSIZE, (uint64)mem, PTE_W|PTE_R|PTE_X|PTE_U);
  memmove(mem, src, sz);
}
```

### `uvmalloc`
调整页表分配的内存，从`old_size`变更到`new_size`  
* 如果分配的物理内存不够用，那么把刚分配到一半的内存释放掉，返回0
* 否则给页表添加上新的映射。

### `uvmdealloc`
缩小用户页表空间

### `freewalk`
递归释放页表；页表的叶子节点上需要全部都释放了map

### `uvmfree`
释放size空间的内存：首先取消映射(`uvmunmap`),然后移除页表（`freewalk`）

### `uvmcopy`
把旧的页表复制到新的页表；复制失败或错误会恢复原状

### `uvmclear`
让一个页表进入`invalid`状态


# `kalloc.c`
`kalloc.c`中主要包含物理内存分配的相关内容  
## `kmem`
```c
struct {
  struct spinlock lock;
  struct run *freelist;
} kmem;
```
维护物理内存的分配状况，其形式是一个链表。它指向的是一个未被使用的页
### `kfree`
把`pa`指向的物理内存释放  
`kfree`会把每个字节置为`0x01`(junk)来检测悬空引用(`dangling refs`)  
`kfree`通过把当前释放的节点放在`freelist`最前面
```c
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  acquire(&kmem.lock);
  r->next = kmem.freelist;
  kmem.freelist = r;
  release(&kmem.lock);
}
```
### `freerange`
把一段物理内存释放出来放进`free_list`中
```c
void
freerange(void *pa_start, void *pa_end)
{
  char *p;
  p = (char*)PGROUNDUP((uint64)pa_start);
  for(; p + PGSIZE <= (char*)pa_end; p += PGSIZE)
    kfree(p);
}
```
### `kalloc`
分配一个物理页  

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
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}
```
# `sbrk`
这是拓展或者缩小内存的系统调用  
调用`syscall sbrk`的时候，调用的是[`kernel/proc.c::growproc()`](kernel/proc.c)
