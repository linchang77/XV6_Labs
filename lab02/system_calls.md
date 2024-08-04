# Lab02: System Calls(写系统调用)

## System Call Tracing (Moderate)

### 1) 实验目的

在这个作业中，你将添加一个系统调用跟踪功能，这可能会在调试后续实验时对你有所帮助。你需要创建一个新的 trace 系统调用，用于控制跟踪。它应该接受一个参数，即一个整数“掩码”，该掩码的位指定要跟踪哪些系统调用。例如，为了跟踪 fork 系统调用，程序应调用 trace(1 << SYS_fork)，其中 SYS_fork 是来自 kernel/syscall.h 的系统调用编号。你需要修改 xv6 内核，以便在每个系统调用即将返回时打印出一行内容，如果系统调用的编号在掩码中被设置。该行应包含进程 ID、系统调用名称和返回值；你不需要打印系统调用的参数。trace 系统调用应为调用它的进程以及随后由它派生的任何子进程启用跟踪，但不应影响其他进程。

### 2) 实验步骤

1. **添加系统调用原型和系统调用号**：
   - 在 `user/user.h` 中添加 `trace` 系统调用的原型：
     ```c
     int trace(int mask);
     ```
   - 在 `kernel/syscall.h` 中添加 `SYS_trace` 的系统调用号：
     ```c
     #define SYS_trace 22
     ```

2. **生成系统调用存根**：
   - 修改 `user/usys.pl`，添加 `trace` 系统调用：
     ```perl
     entry("trace");
     ```

3. **实现 `trace` 系统调用**：
   - 在 `kernel/sysproc.c` 中实现 `sys_trace` 函数：
     ```c
     int sys_trace(void) {
       int mask;
       if (argint(0, &mask) < 0)
         return -1;
       myproc()->tracemask = mask;
       return 0;
     }
     ```

4. **修改 `fork` 函数**：
   - 在 `kernel/proc.c` 中，确保子进程继承父进程的追踪掩码：
     ```c
     np->tracemask = p->tracemask;
     ```

5. **修改 `syscall` 函数**：
   - 在 `kernel/syscall.c` 中，添加系统调用名称数组：
     ```c
     static char *syscallnames[] = {
       // ... existing syscall names ...
       [SYS_trace] "trace",
     };
     ```
   - 修改 `syscall` 函数以打印追踪信息：
     ```c
     void syscall(void) {
       int num;
       struct proc *p = myproc();

       num = p->trapframe->a7;
       if (num > 0 && num < NELEM(syscalls) && syscalls[num]) {
         p->trapframe->a0 = syscalls[num]();
         if (p->tracemask & (1 << num)) {
           printf("%d: syscall %s -> %d\n", p->pid, syscallnames[num], p->trapframe->a0);
         }
       } else {
         printf("%d %s: unknown sys call %d\n", p->pid, p->name, num);
         p->trapframe->a0 = -1;
       }
     }
     ```

6. **编译和运行**：
   - 在 `Makefile` 中将 `trace` 添加到 `UPROGS` 中：
  
     ```makefile
     UPROGS = ... _trace ...
     ```

   - 运行 `make qemu` 并测试 `trace` 程序：

     ```sh
     $ trace 32 grep hello README
     3: syscall read -> 1023
     3: syscall read -> 966
     3: syscall read -> 70
     3: syscall read -> 0
     ```

### 3) 实验中遇到的困难和解决办法

1. **理解The 32 is 1<<SYS_read.**：
   - **困难描述**：无法看懂trace的使用方法，在提示中出现1<<SYS_read。
   - **解决办法**：查找资料得知<<是位移运算，1<<SYS_read就是把1左移5位（文件中SYS_read=5）得到十进制32
   思考：用掩码的形式可以充分利用计算机采用二进制的性质，方便代码中```(1<<num)&(p->tracemask)```这样的比较。

1. **理解系统调用的复杂过程**：
   - **困难描述**：实现一个系统调用需要修改多个文件，并且理解他们的含义。
   - **解决办法**：认真阅读提示并查阅资料了解到需要在 user/user.h 中添加系统调用的原型，在 user/usys.pl 中添加存根，以及在 kernel/syscall.h 中添加系统调用编号。

### 4) 实验心得

通过本次实验，我深入理解了 xv6 操作系统中系统调用的实现过程（在解决方法里有），并掌握了如何通过添加自定义系统调用来扩展操作系统的功能。实验中遇到的问题让我更加熟悉了系统调用的参数传递和进程间通信机制。在调试过程中，我学会了如何利用系统调用追踪功能来排查问题，这对后续的实验和项目开发具有重要意义。整体上，这次实验不仅巩固了我对操作系统课程理论知识的理解，也提升了我在实际操作系统开发中的实践能力。


## Sysinfo 系统调用(Moderate)

### 1）实验目的

在这个任务中，你需要添加一个系统调用 sysinfo，用于收集关于正在运行的系统的信息。这个系统调用接受一个参数：指向 struct sysinfo 的指针（见 kernel/sysinfo.h）。内核应填写这个结构体的各个字段：freemem 字段应设置为空闲内存的字节数，nproc 字段应设置为状态不是 UNUSED 的进程数量。我们提供了一个测试程序 sysinfotest；如果它打印出 "sysinfotest: OK"，则表示你完成了这个任务。

### 2）实验步骤

1. **添加系统调用声明和用户空间函数**
   - 在 `user/user.h` 中声明 `sysinfo` 系统调用。由于我们需要使用 `sysinfo` 结构体，因此需要提前声明：

     ```c
     struct sysinfo;
     int sysinfo(struct sysinfo *);
     ```

   - 在 `user/usys.pl` 中添加 `sysinfo` 的声明以生成系统调用的汇编代码。

2. **定义 sysinfo 结构体**
   - 在 `kernel/sysinfo.h` 中定义 `sysinfo` 结构体，该结构体包含两个字段：`freemem` 和 `nproc`。

     ```c
     struct sysinfo {
         uint freemem;
         uint nproc;
     };
     ```

3. **实现 sysinfo 系统调用**
   - 在 `kernel/sysproc.c` 中实现 `sysinfo` 系统调用。该调用需要填充 `sysinfo` 结构体，并将其拷贝到用户空间。

     ```c
     extern uint64 freemem();
     extern uint64 proc_count();

     uint64
     sys_sysinfo(void)
     {
         struct sysinfo info;
         uint64 addr;

         if(argaddr(0, &addr) < 0)
             return -1;

         info.freemem = freemem();
         info.nproc = proc_count();

         if(copyout(myproc()->pagetable, addr, (char *)&info, sizeof(info)) < 0)
             return -1;

         return 0;
     }
     ```

4. **收集系统信息**
   - **收集空闲内存**：
     在 `kernel/kalloc.c` 中添加 `freemem` 函数，该函数计算系统中当前空闲的内存量。

     ```c
     uint64
     freemem(void)
     {
         struct run *r;
         uint64 free = 0;

         acquire(&kmem.lock);
         for(r = kmem.freelist; r; r = r->next)
             free += PGSIZE;
         release(&kmem.lock);

         return free;
     }
     ```

   - **收集进程数量**：
     在 `kernel/proc.c` 中添加 `proc_count` 函数，该函数计算非 `UNUSED` 状态的进程数量。

     ```c
     uint64
     proc_count(void)
     {
         struct proc *p;
         int count = 0;

         for(p = proc; p < &proc[NPROC]; p++) {
             if(p->state != UNUSED)
                 count++;
         }

         return count;
     }
     ```

5. **测试和调试**
   - 修改 Makefile，确保 `sysinfotest` 已添加到 `UPROGS` 中。
   - 编译并运行 xv6，在 xv6 shell 中执行 `sysinfotest` 以验证实现是否正确。

     ```sh
     make clean
     make qemu
     ```

### 3）实验中遇到的困难和解决办法

1. **如何将 struct sysinfo 复制回用户空间**
   - **困难描述**：sysinfo 需要将 内核态中获取的struct sysinfo 复制回用户空间。
   - **解决办法**：根据提示阅读源码sys_fstat()（kernel/sysfile.c）和 filestat()（kernel/file.c），明白了如何使用 copyout() 来实现这一点。

### 4）实验心得

通过本次实验，我深入理解了 xv6 操作系统中系统调用的实现过程。添加一个新的系统调用需要在多个文件中进行修改和注册，包括用户空间和内核空间的函数声明、系统调用号的分配、实际功能的实现以及返回结果的拷贝等。通过实现 `sysinfo` 系统调用，我不仅熟悉了 xv6 中内存和进程管理的基本机制，还掌握了如何在内核中收集和返回系统信息。此外，在调试过程中，我学会了如何处理和解决常见的编译和运行时错误，这为我将来进行更多复杂的操作系统开发打下了坚实的基础。



