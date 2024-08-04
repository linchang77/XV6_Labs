# Lab06: Copy-on-Write Fork for xv6

## Implement Copy-On-Write (Hard)

### 1) 实验目的

本实验的目的是在 xv6 中实现写时复制（Copy-On-Write, COW）机制，从而优化 `fork()` 系统调用的内存使用。通过实现 COW，我们可以延迟物理内存的分配，直到写操作发生。目标是使得内核能够通过 `cowtest` 和 `usertests` 的所有测试。

COW（写时复制）fork() 的目标是推迟为子进程分配和复制物理内存页，直到实际需要复制时才进行。

COW fork() 仅为子进程创建一个页表，用户内存的页表项（PTEs）指向父进程的物理页面。COW fork() 将父进程和子进程中的所有用户页表项标记为不可写。当任一进程尝试写入这些COW页面之一时，CPU会强制产生一个页面错误。内核的页面错误处理程序检测到这种情况，为发生错误的进程分配一个物理页面，将原始页面复制到新页面，并修改发生错误进程中的相关页表项以引用新页面，这次将页表项标记为可写。当页面错误处理程序返回时，用户进程将能够写入其页面副本。

COW fork() 使实现用户内存的物理页面的释放变得更复杂。一个给定的物理页面可能被多个进程的页表引用，只有当最后一个引用消失时，才应该释放该物理页面。

### 2) 实验步骤

1. **修改 `uvmcopy()` 以支持 COW**：
   - **在 `kernel/vm.c` 中修改 `uvmcopy()`**，使其在 `fork()` 时将父进程的物理页面映射到子进程，并清除 `PTE_W` 标志：
     ```c
     int
     uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
     {
         uint64 i, pa;
         pte_t *pte;
         char *mem;
         
         for(i = 0; i < sz; i += PGSIZE){
             pte = walk(old, i, 0);
             if(pte == 0 || !(*pte & PTE_V))
                 panic("uvmcopy: pte should exist");

             pa = PTE2PA(*pte);
             if(uvmmap(new, i, pa, PGSIZE, PTE_R | PTE_X | PTE_U | PTE_COW) != 0)
                 return -1;
         }
         return 0;
     }
     ```

2. **修改 `usertrap()` 以处理 COW 页面错误**：
   - **在 `kernel/trap.c` 中修改 `usertrap()`**，处理 COW 页面错误：
     ```c
     void
     usertrap(void)
     {
         uint64 va;
         pte_t *pte;
         struct proc *p = myproc();

         // 检查是否是页面错误
         if (r_scause() == 13 || r_scause() == 15) {
             va = r_stval();
             va = PGROUNDDOWN(va);
             
             pte = walk(p->pagetable, va, 0);
             if (pte && (*pte & PTE_COW)) {
                 char *mem = kalloc();  //为子进程分配空间
                 if (mem == 0) {
                     printf("usertrap(): out of memory\n");
                     exit(-1);
                 }
                 memmove(mem, (char*)PTE2PA(*pte), PGSIZE);
                 if (mappages(p->pagetable, va, PGSIZE, (uint64)mem, PTE_R | PTE_W | PTE_U) != 0) {
                     printf("usertrap(): mappages failed\n");
                     exit(-1);
                 }
                 *pte = PA2PTE((uint64)mem) | PTE_R | PTE_W | PTE_U;
                 sfence_vma();
                 return;
             }
         }
         // 处理其他异常
         // ...
     }
     ```

3. **在 `kalloc()` 中实现物理页的引用计数**：
   - **在 `kernel/kalloc.c` 中添加引用计数机制**：
     ```c
     #define MAX_PHYS_PAGES 1024
     static int refcount[MAX_PHYS_PAGES];

     void
     kinit(void)
     {
         // 初始化物理页面
         memset(refcount, 0, sizeof(refcount));
     }

     char*
     kalloc(void)
     {
         // 分配页面并更新引用计数
         char *p = kalloc_impl();  // 你需要实现这个函数
         if (p) {
             int idx = PTE2PA(p) / PGSIZE;
             refcount[idx] = 1;
         }
         return p;
     }

     void
     kfree(char *p)
     {
         int idx = PTE2PA(p) / PGSIZE;
         if (--refcount[idx] == 0) {
             kfree_impl(p);  // 你需要实现这个函数
         }
     }
     ```

4. **修改 `copyout()` 以支持 COW**：
   - **在 `kernel/sysproc.c` 中修改 `copyout()`**，处理 COW 页面：
    在遇到COW页面时使用与页面错误相同的方案
     ```c
     int
     copyout(pagetable_t pagetable, uint64 va, void *p, uint64 len)
     {
         uint64 i;

         for(i = 0; i < len; i += PGSIZE){
             if (walk(pagetable, i, 0) && (*walk(pagetable, i, 0) & PTE_COW)) {
                 // Handle COW pages
                 if (mappages(pagetable, va + i, PGSIZE, PA2PTE(PTE2PA(*walk(pagetable, i, 0))), PTE_R | PTE_W | PTE_U) != 0)
                     return -1;
             }
         }

         return copyout_impl(pagetable, va, p, len);
     }
     ```

5. **测试和验证**：
   - **编译并运行 `cowtest`**，确保所有测试通过。
   - **编译并运行 `usertests`**，确保所有测试通过。
   - **检查输出是否符合预期**，验证内核的稳定性和 COW 功能的正确性。

### 3) 实验中遇到的困难和解决办法

1. **copy-on-write原理的理解**：
   - **困难描述**：这个实验的核心就是完成copy-on-write功能。
   - **解决办法**：上网查找资料。推迟为子进程分配和复制物理内存页，直到实际需要复制时才进行，以达到更好的利用存储空间的目的

2. **COW 释放物理页面**：
   - **困难描述**：要求中需要确保在最后一个页表项引用消失时释放每个物理页面——但不能提前释放。
   - **解决办法**：查找资料发现可以为每个物理页面维护一个“引用计数”，记录引用该页面的用户页表数量。当kalloc()分配页面时，将该页面的引用计数设置为1。当fork导致子进程共享页面时，增加页面的引用计数；每次任何进程从其页表中删除该页面时，减少页面的引用计数。只有当引用计数为零时，kfree()才应将页面放回自由列表。。

### 4) 实验心得

通过本次实验，我深入理解了写时复制（COW）机制在操作系统中的应用，特别是在内存管理和进程复制方面的优化。实现 COW 使得内存使用更加高效，避免了不必要的内存复制，提高了系统的性能。解决实验中的问题让我提高了对内核内存管理和系统调用的理解，也增强了调试和问题解决的能力。