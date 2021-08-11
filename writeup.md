这几个lab都不难，主要是涉及的代码文件比较多，需要明确它们的作用。

# System call tracing
第一个lab是添加syscall的trace   
提示
* 添加`$U/_trace`

* 运行`make qemu`，会提示无法编译；需要添加`syscall trace`。具体的步骤是：
	* `user/user.h`里加上`int sysinfo (struct sysinfo *);`
	* `user/usys.pl`加上`entry("sysinfo");`
	* `kernel/syscall.h`加上`#define SYS_sysinfo 23`
* 这样就有`syscall trace`了。此时还没有具体的实现，需要在`kernel/sysproc.c`里添加一个函数；但是这样系统仍然不知道`syscall trace`对应的是哪一个函数，需要添加一个函数指针。在`kernel/sysproc.c`的函数指针数组`static uint64 (*syscalls[]) (void)`中，加上一项`[SYS_sysinfo] sys_sysinfo`。注意到`sys_sysinfo`会报错，原因是没有声明。这是因为这个文件没有include`sysproc.c`的所有函数，需要在`kernel/defs.h`里加上`sys_sysinfo`的声明。

* 需要修改`kernel/proc.c的fork()`，使得子进程也有trace属性

* 修改`syscall()`以打印trace结果

提示大致把添加`syscall`的过程告诉我们了，按上面的流程做一遍差不多就行了。其中`sysproc.c`下的函数`sys_trace`如下：
```c
uint64
sys_trace (void)
{
	return 0;
}
```

接下来要添加sys_trace的内容。首先看用户调用`syscall trace`的例程(`user/trace.c`)
```c
int
main(int argc, char *argv[])
{
  int i;
  char *nargv[MAXARG];

  if(argc < 3 || (argv[1][0] < '0' || argv[1][0] > '9')){
    fprintf(2, "Usage: %s mask command\n", argv[0]);
    exit(1);
  }

  if (trace(atoi(argv[1])) < 0) {
    fprintf(2, "%s: trace failed\n", argv[0]);
    exit(1);
  }
  
  for(i = 2; i < argc && i < MAXARG; i++){
    nargv[i-2] = argv[i];
  }
  exec(nargv[0], nargv);
  exit(0);
}
```
可以看到，trace函数实际上不是立刻生效的，而是只打一个标记，并返回0。我们只需要把传进来的`mask`挂在进程上就行了。  
参考`sysproc`上面的例子：
```c
uint64
sys_trace (void)
{
  int trace_calls;

  if (argint (0, &trace_calls) < 0)
    return -1;
  myproc ()->trace = trace_calls;
  return 0;
}
```
在`proc.h`中结构体`proc`中也要加上一项`trace`


trace需要在`syscall`时打印一些内容，因此看`syscall()`函数：
```c
void
syscall (void)
{
  int num;
  struct proc *p = myproc (); //获取进程
  num = p->trapframe->a7;
  if (num > 0 && num < NELEM (syscalls) && syscalls[num]) {
    p->trapframe->a0 = syscalls[num]();
  } else {
    printf ("%d %s: unknown sys call %d\n", p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
  if ((1 << num) & p->trace) { // 如果打开了trace，并且trace的是当前syscall
    printf ("%d: syscall %s -> %d\n", p->pid, syscalls_name[num],
            p->trapframe->a0);
  }
}
```
因为没有syscall对应的字符，我们还需要自己加上debug的字符：
```c
static const char *syscalls_name[32] = {
  "unknow", "fork",  "exit",   "wait",   "pipe",  "read",  "kill",   "exec",
  "fstat",  "chdir", "dup",    "getpid", "sbrk",  "sleep", "uptime", "open",
  "write",  "mknod", "unlink", "link",   "mkdir", "close", "trace",  "sysinfo",
};
```
接下来是细节问题：
* 进程结束后，trace标记怎么去掉；子进程怎么拷贝父进程的标记  
	看`proc.c中的allocproc()`可以发现，进程是被不断复用的，我们的`trace`标记不会被清掉，另一方面，进程似乎都是通过`fork`进行复制的，因此只需要在fork的最后面修改标记就行了。
	```c
	// .... fork()
  safestrcpy (np->name, p->name, sizeof (p->name));

  pid = np->pid;

  np->state = RUNNABLE;
  //--------------- 加上的部分
  if (p->trace) { 
    // set trace
    np->trace = p->trace;
  } else {
    np->trace = 0;
  }
  //------------------
  release (&np->lock);
	
  return pid;
}
	```


	