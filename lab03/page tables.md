# Lab03: Page Tables(关于页表)

## Print a Page Table (Easy)

### 1) 实验目的

定义一个名为 vmprint() 的函数。该函数应接受一个 pagetable_t 类型的参数，并以如下所述的格式打印该页表。在 exec.c 文件中，在 return argc 之前插入 if(p->pid==1) vmprint(p->pagetable)，以打印第一个进程的页表。如果通过 make grade 的 PTE 打印测试，你将获得此任务的满分。

### 2) 实验步骤

1. **定义 `vmprint` 函数**：
   - 在 `kernel/vm.c` 文件中实现 `vmprint` 函数。

2. **声明 `vmprint` 函数原型**：
   - 在 `kernel/defs.h` 中添加 `vmprint` 函数的原型：
     ```c
     void vmprint(pagetable_t pagetable);
     ```

3. **实现 `vmprint` 函数**：
   - 在 `kernel/vm.c` 文件中添加以下代码：
     ```c
     void
     vmprint(pagetable_t pagetable)
     {
         printf("page table %p\n", pagetable);
         vmprint_rec(pagetable, 0);
     }

     void
     vmprint_rec(pagetable_t pagetable, int level)
     {
         for (int i = 0; i < 512; i++) {
             pte_t pte = pagetable[i];
             if (pte & PTE_V) {
                 uint64 pa = PTE2PA(pte);
                 for (int j = 0; j < level; j++) {
                     printf("..");
                 }
                 printf("%d: pte %p pa %p\n", i, pte, pa);
                 if ((pte & (PTE_R | PTE_W | PTE_X)) == 0) {
                     pagetable_t next_level = (pagetable_t)pa;
                     vmprint_rec(next_level, level + 1);
                 }
             }
         }
     }
     ```

4. **修改 `exec` 函数**：
   - 在 `kernel/exec.c` 中，确保在返回前打印出第一个进程的页表：
     ```c
     if (p->pid == 1) {
         vmprint(p->pagetable);
     }
     ```

5. **编译和运行**：
   - 使用 `make qemu` 重新编译并运行 xv6，检查输出是否符合预期格式。

### 3) 实验中遇到的困难和解决办法

1. **如何打印页表**：
   - **困难描述**：这个实验要求打印出第一个开始进程的页表。
   - **解决办法**：查看 freewalk 函数，并且上网了解xv6页表存储的模式，采用多级页表存储，需要递归的打印页表信息。
2. **理解xv6页表项含义**
   - **要想做这些实验必须了解页表含义**
   - **解决办法**：阅读教材：
PTE 的组成部分
Valid 位（PTE_V）：

表示这个页表项是否有效。如果这个位被设置为1，说明这个页表项有效；如果为0，则无效。
Read 位（PTE_R）：

表示这个页表项所指向的页面是否可读。如果这个位被设置为1，说明该页面可读。
Write 位（PTE_W）：

表示这个页表项所指向的页面是否可写。如果这个位被设置为1，说明该页面可写。
Execute 位（PTE_X）：

表示这个页表项所指向的页面是否可执行。如果这个位被设置为1，说明该页面可执行。
User 位（PTE_U）：

表示这个页表项所指向的页面是否可以在用户模式下访问。如果这个位被设置为1，说明该页面在用户模式下也可以访问。
Global 位（PTE_G）：

表示这个页表项是否为全局页。全局页在上下文切换时不会从 TLB 中移除。
Accessed 位（PTE_A）：

表示这个页是否已被访问。如果这个位被设置为1，说明该页在某个时刻被访问过。
Dirty 位（PTE_D）：

表示这个页是否被写过。如果这个位被设置为1，说明该页在某个时刻被写过。
物理页地址（Physical Page Number，PPN）：

表示这个页表项所映射的物理页的地址。通常物理页地址占据了 PTE 的高位部分。

### 4) 实验心得

通过本次实验，我深入理解了 RISC-V 页表的结构和工作原理。实现 `vmprint` 函数不仅帮助我熟悉了页表的遍历和递归处理，还让我学会了如何利用打印输出调试复杂的数据结构。实验过程中遇到的问题让我更加熟悉了 PTE 的解析和页表的层级关系。在调试过程中，我学会了如何通过逐步检查输出来排查问题，这对后续的实验和项目开发具有重要意义。整体上，这次实验不仅巩固了我对操作系统课程理论知识的理解，也提升了我在实际操作系统开发中的实践能力。

## A Kernel Page Table Per Process (Hard)

### 1) 实验目的

你的第一个任务是修改内核，使得每个进程在内核中执行时使用自己的内核页表副本。修改 struct proc 以维护每个进程的内核页表，并修改调度器在切换进程时切换内核页表。对于这一步，每个进程的内核页表应该与现有的全局内核页表相同。如果 usertests 能正确运行，你就通过了这一部分的实验。

### 2) 实验步骤

1. **修改 `struct proc`**：
   - 在 `kernel/proc.h` 中添加一个字段，用于存储每个进程的内核页表：
     ```c
     struct proc {
       // existing fields...
       pagetable_t kpagetable;  // Kernel page table for this process
     };
     ```

2. **初始化进程的内核页表**：
   - 在 `allocproc` 中为新进程创建内核页表：
     ```c
     pagetable_t
     proc_kvminit(void)
     {
         pagetable_t pagetable;
         
         // Allocate root page-table
         pagetable = (pagetable_t) kalloc();
         if(pagetable == 0)
             return 0;
         memset(pagetable, 0, PGSIZE);

         // Copy kernel mappings
         if(kvmmap(pagetable, UART0, UART0, PGSIZE, PTE_R | PTE_W) < 0)
             goto bad;
         if(kvmmap(pagetable, VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W) < 0)
             goto bad;
         if(kvmmap(pagetable, CLINT, CLINT, 0x10000, PTE_R | PTE_W) < 0)
             goto bad;
         if(kvmmap(pagetable, PLIC, PLIC, 0x400000, PTE_R | PTE_W) < 0)
             goto bad;
         if(kvmmap(pagetable, KERNBASE, KERNBASE, (uint64)etext - KERNBASE, PTE_R | PTE_X) < 0)
             goto bad;
         if(kvmmap(pagetable, (uint64)etext, (uint64)etext, PHYSTOP - (uint64)etext, PTE_R | PTE_W) < 0)
             goto bad;

         return pagetable;

       bad:
         proc_kvmunmap(pagetable, 0, PGSIZE, 1);
         return 0;
     }
     ```

3. **分配和释放内核页表**：
   - 在 `allocproc` 中调用 `proc_kvminit` 为新进程分配内核页表：
     ```c
     p->kpagetable = proc_kvminit();
     if (p->kpagetable == 0) {
         freeproc(p);
         release(&p->lock);
         return 0;
     }
     ```

   - 在 `freeproc` 中释放进程的内核页表：
     ```c
     if (p->kpagetable) {
         proc_kvmunmap(p->kpagetable, 0, PGSIZE, 1);
         kfree((void*)p->kpagetable);
         p->kpagetable = 0;
     }
     ```

4. **修改调度器**：
   - 在 `scheduler` 函数中加载进程的内核页表到 SATP 寄存器中：
     ```c
     void
     scheduler(void)
     {
         struct proc *p;
         for(;;){
             // Avoid deadlock by ensuring that devices can interrupt.
             intr_on();

             for(p = proc; p < &proc[NPROC]; p++){
                 acquire(&p->lock);
                 if(p->state == RUNNABLE){
                     // Switch to chosen process. It is the process's job
                     // to release its lock and then reacquire it
                     // before jumping back to us.
                     p->state = RUNNING;

                     w_satp(MAKE_SATP(p->kpagetable));//使用 w_satp 设置进程的页表
                     sfence_vma();//调用 sfence_vma 刷新虚拟内存地址

                     swtch(&c->scheduler, &p->context);//调用 swtch 函数切换到进程 p 的上下文。c->scheduler 是调度器的上下文，而 p->context 是要切换到的进程的上下文。

                     w_satp(MAKE_SATP(kernel_pagetable));//切换回内核页表，并刷新虚拟内存地址。
                     sfence_vma();

                     // Process is done running for now.
                     // It should have changed its p->state before coming back.
                 }
                 release(&p->lock);
             }
         }
     }
     ```

### 3) 实验中遇到的困难和解决办法

1. **页表映射问题**：
   - **困难描述**：如何将页表的虚拟地址和真实地址进行有效转换。
   - **解决办法**：阅读xv6book以及网上资料，了解到了xv6采用直接地址映射方式，需要使用kvmmap函数进行映射，并且需要映射六个地址。

2. **SATP切换问题**：
   - **困难描述**：由于设置了内置的页表，所以需要在原先的调度器中进行页表切换。
   - **解决办法**：参见 kvminithart 获得灵感，在 `scheduler` 函数中正确调用 `w_satp` 和 `sfence_vma`，确保页表切换后立即生效。



### 4) 实验心得

通过本次实验，我深入理解了操作系统中页表的管理和切换机制。为每个进程维护独立的内核页表，不仅提高了系统的安全性和隔离性，也为后续实验中内核直接解引用用户指针打下了基础。实验过程中遇到的映射问题和页表切换问题让我更加熟悉了页表的初始化和管理。在调试过程中，我学会了如何通过打印页表内容来排查问题，这对后续的实验和项目开发具有重要意义。整体上，这次实验不仅巩固了我对操作系统课程理论知识的理解，也提升了我在实际操作系统开发中的实践能力。

## Simplify `copyin`/`copyinstr` (Hard)

### 1) 实验目的

本实验的目的是将 kernel/vm.c 中 copyin 的主体替换为调用 kernel/vmcopyin.c 中定义的 copyin_new；对 copyinstr 和 copyinstr_new 也做同样的操作。为每个进程的内核页表添加用户地址映射，使 copyin_new 和 copyinstr_new 能够正常工作。
设计了一种将用户态页表塞到内核页表中的机制。 如此一来， 当已进入内核的进程想要查询进程
用户态的某一地址空间时， 无需再切回用户态， 直接在内核中就能实现查询和地
址翻译工作， 这样大大节省了 user 和 kernel 之间来来回回切换的代价。

### 2) 实验步骤

1. **替换 `copyin` 和 `copyinstr`**：
   - 在 `kernel/vm.c` 中将 `copyin` 替换为 `copyin_new` 的调用：
     ```c
     int
     copyin(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len)
     {
         return copyin_new(pagetable, dst, srcva, len);
     }
     ```
   - 将 `copyinstr` 替换为 `copyinstr_new` 的调用：
     ```c
     int
     copyinstr(pagetable_t pagetable, char *dst, uint64 srcva, uint64 max)
     {
         return copyinstr_new(pagetable, dst, srcva, max);
     }
     ```

2. **更新内核页表映射**：
   - 在 `allocproc` 中为新进程的内核页表添加用户地址映射：
     ```c
     if (mappages(p->kpagetable, TRAMPOLINE, PGSIZE, (uint64)trampoline, PTE_R | PTE_X) < 0 ||
         mappages(p->kpagetable, TRAPFRAME, PGSIZE, (uint64)(p->trapframe), PTE_R | PTE_W) < 0)
     {
         proc_kvmunmap(p->kpagetable, 0, PGSIZE, 1);
         freeproc(p);
         release(&p->lock);
         return 0;
     }
     ```

3. **修改 `fork`**：
   - 在 `fork` 中添加用户地址映射：
     ```c
     if (uvmcopy(p->pagetable, np->pagetable, p->sz) < 0) {
         freeproc(np);
         release(&np->lock);
         return -1;
     }

     np->sz = p->sz;

     if (kvmmap(np->kpagetable, TRAMPOLINE, PGSIZE, (uint64)trampoline, PTE_R | PTE_X) < 0 ||
         kvmmap(np->kpagetable, TRAPFRAME, PGSIZE, (uint64)(np->trapframe), PTE_R | PTE_W) < 0) {
         freeproc(np);
         release(&np->lock);
         return -1;
     }
     ```

4. **修改 `exec`**：
   - 在 `exec` 中添加用户地址映射：
     ```c
     if ((sz = uvmalloc(pagetable, 0, sz)) == 0)
         goto bad;

     if (kvmmap(p->kpagetable, TRAMPOLINE, PGSIZE, (uint64)trampoline, PTE_R | PTE_X) < 0 ||
         kvmmap(p->kpagetable, TRAPFRAME, PGSIZE, (uint64)(p->trapframe), PTE_R | PTE_W) < 0)
         goto bad;
     ```

5. **修改 `sbrk`**：
   - 在 `sbrk` 中添加用户地址映射：
     ```c
     if (n > 0) {
         if ((addr = uvmalloc(p->pagetable, oldsz, oldsz + n)) == 0)
             return -1;
     } else if (n < 0) {
         if ((addr = uvmdealloc(p->pagetable, oldsz, oldsz + n)) == 0)
             return -1;
     }

     p->sz = addr;
     ```

### 3) 实验中遇到的困难和解决办法

1. **内核页表映射问题**：
   - **困难描述**：在添加用户地址映射时，可能会遇到映射错误或缺失映射，导致内存访问异常。
   - **解决办法**：仔细检查页表映射代码，确保所有必须的用户地址都正确映射到内核页表中。

2. **PLIC 限制限制问题**：
   - **困难描述**：要求中说用户虚拟地址范围不与内核用于其自身指令和数据的虚拟地址范围重叠。Xv6 使用从零开始的虚拟地址作为用户地址空间，而内核的内存幸运地从更高的地址开始。然而，这种方案限制了用户进程的最大大小，必须小于内核的最低虚拟地址。在内核引导后，在xv6中这个地址是0xC000000，即PLIC寄存器的地址；。
   - **解决办法**：我一开始没有注意到这个限制，导致测试的时候会报错，后来查了相关资料，在相关代码中增加检查，确保用户进程的大小不会超过内核的最低虚拟地址 (0xC000000)。
### 5) 实验心得

通过本次实验，我深入理解了如何通过为每个进程的内核页表添加用户地址映射来简化 `copyin` 和 `copyinstr` 的实现。实验过程中遇到的页表映射问题和用户地址空间限制问题让我更加熟悉了内存管理和页表操作。在调试过程中，我学会了如何通过打印页表内容来排查问题，这对后续的实验和项目开发具有重要意义。整体上，这次实验不仅巩固了我对操作系统课程理论知识的理解，也提升了我在实际操作系统开发中的实践能力。
