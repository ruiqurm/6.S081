# Print a page table
提示：  
* `vmprint()`放在kernel/vm.c.
* 可以用`kernel/riscv.h`末尾的几个宏  
* `vm.c`里的`freewalk`可以参考
*  在`kernel/defs.h`声明`vmprint`，这样才能在`exec.c`中使用  
* 使用`%p`打印64bit指针

这个lab跟着hint做就行了,参照`freewalk`，我写的是这样的：
```c
void
vmprint(pagetable_t pagetable,int level){
  if(level==0){
    printf("page table %p\n",pagetable);
  }
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    uint64 child = PTE2PA(pte); // 物理地址
    if(pte & PTE_V){
      for(int j =0;j<=level;j++){
        printf("..");
      }
      printf("%d: pte %p pa %p\n",i,pte,child);
      if((pte & (PTE_R|PTE_W|PTE_X)) == 0){
        // 非叶子，继续递归
        vmprint((pagetable_t)child,level+1);
      }
    }
  }
}
```
然后在`kernel/defs.h`添加声明；在`exec.c`里添加该函数。调用函数时，页表必须全部生成完，因此我是在旧页表替换的时候添加的：
```c
  //....
  for(last=s=path; *s; s++)
    if(*s == '/')
      last = s+1;
  safestrcpy(p->name, last, sizeof(p->name));
  vmprint(pagetable,0); //打印页表
  oldpagetable = p->pagetable;
  p->pagetable = pagetable;
  // ...
```

# A kernel page table per process
这个lab的测试似乎有点问题，我第一遍写的时候其实并没有符合要求，然而还是过了.后面看了别人的写法才发现自己写的不对。  
