# Lab08: locks

以下是关于 xv6 内存分配器改进的实验报告：

---

## 实验报告：多核环境下的内存分配器优化

### 1. 实验目的

本实验旨在优化 xv6 操作系统的内存分配器，以减少多核环境下的锁争用现象。原有的内存分配器使用一个全局自由列表（free list），由单个锁保护，导致在高并发情况下产生严重的锁争用问题。通过实现每个 CPU 一个自由列表，并在需要时允许 CPU 之间窃取（steal）内存块，可以提高并行度，降低锁争用，从而提升系统性能。

kalloctest 中锁争用的根本原因是 kalloc() 有一个单独的空闲列表，由一个锁保护。为了消除锁争用，你需要重新设计内存分配器，以避免单一的锁和列表。基本思路是为每个 CPU 维护一个空闲列表，每个列表有自己的锁。不同 CPU 上的分配和释放可以并行运行，因为每个 CPU 都在操作不同的列表。

### 3. 实验步骤

#### 2. 修改内存分配器

1. **分配每个 CPU 一个自由列表**：
   在 `kalloc.c` 中，为每个 CPU 创建一个独立的自由列表和对应的锁。

   ```c
   struct {
       struct spinlock lock;
       struct run *freelist;
   } kmem[NCPU];
   ```

2. **修改 `freerange` 函数**：
   将所有空闲内存块分配给当前运行 `freerange` 的 CPU。

   ```c
   void freerange(void *vstart, void *vend) {
       char *p;
       p = (char*)PGROUNDUP((uint64)vstart);
       for(; p + PGSIZE <= (char*)vend; p += PGSIZE)
           kfree(p);
   }
   ```

3. **修改 `kalloc` 和 `kfree` 函数**：
   - `kalloc` 从当前 CPU 的自由列表中分配内存，如果当前列表为空，则尝试从其他 CPU 窃取内存。
   - `kfree` 将释放的内存块加入当前 CPU 的自由列表。

   ```c
   void kfree(void *pa) {
       struct run *r = (struct run*)pa;
       push_off();
       int id = cpuid();
       acquire(&kmem[id].lock);
       r->next = kmem[id].freelist;
       kmem[id].freelist = r;
       release(&kmem[id].lock);
       pop_off();
   }

   void* kalloc(void) {
       push_off();
       int id = cpuid();
       acquire(&kmem[id].lock);
       struct run *r = kmem[id].freelist;
       if(r)
           kmem[id].freelist = r->next;
       release(&kmem[id].lock);
       if(r) {
           pop_off();
           memset((char*)r, 5, PGSIZE); // fill with junk
           return (void*)r;
       }

       // Try to steal from other CPUs
       for(int i = 0; i < NCPU; i++) {
           if(i == id) continue;
           acquire(&kmem[i].lock);
           r = kmem[i].freelist;
           if(r) {
               kmem[i].freelist = r->next;
               release(&kmem[i].lock);
               pop_off();
               memset((char*)r, 5, PGSIZE); // fill with junk
               return (void*)r;
           }
           release(&kmem[i].lock);
       }
       pop_off();
       return 0;
   }
   ```


### 3. 遇到的困难

这个实验目标是把每个cpu设置一个空闲列表，同时要求当自己的空闲列表不够的时候从其他cpu的空闲列表获取，然后如何获取我不是很清楚，查看了他那个books，知道可以允许不同
CPU 之间共享内存池的方式可以很好解决这个问题， 首先， 获取当前 CPU 的 ID。
尝试从当前 CPU 的空闲列表中获取空闲块， 如果成功则直接返回。如果当前 CPU
的空闲列表为空， 则遍历其他所有 CPU 的空闲列表， 尝试借用其中的空闲块。
如果找到了空闲块， 则将其从其他 CPU 的空闲列表中移除， 并加入到当前 CPU
的空闲列表中， 然后返回。

### 4. 实验结果

改进后的内存分配器显著减少了 `kmem` 锁的争用，具体表现为 `#fetch-and-add` 值大幅降低。以下是实验前后的对比：

- 实验前 `kmem` 锁争用次数：83375
- 实验后 `kmem` 锁争用次数：0

所有用户测试均通过，表明改进后的内存分配器在性能提升的同时，仍能正确执行内存分配和释放操作。

### 5. 实验总结

通过本次实验，我实现了一个多核环境下的高效内存分配器，解决了单锁引起的高并发争用问题。实验表明，通过将自由列表分配到每个 CPU 并在需要时允许窃取内存块，可以有效降低锁争用，提高系统性能。这种设计思路对多核系统中其他资源的管理也有一定的参考价值。

## Buffer cache (hard)

### 1) 实验目的

本实验的目标是优化 xv6 操作系统中的缓冲区缓存管理，通过减少锁竞争来提高性能。具体来说，你需要实现一个更细粒度的锁机制，用于管理缓冲区缓存，从而减少对 `bcache.lock` 的争用。

### 2) 实验步骤

#### 1. 理解现有代码

1. **查看 `bio.c` 代码**：
   - 定位 `bcache` 结构体以及 `bget` 和 `brelse` 函数。
   - 理解当前 `bcache.lock` 的使用方式及其对性能的影响。

2. **分析性能问题**：
   - 运行 `bcachetest`，查看 `bcache.lock` 的高竞争情况。
   - 记录锁竞争统计信息，识别主要的瓶颈。

#### 2. 实现高级锁机制

1. **定义桶锁**：
   - 实现一个哈希表来管理缓冲区缓存，每个桶使用一个锁。使用质数作为桶的数量（例如：13）来减少哈希冲突。

   ```c
   #define NUM_BUCKETS 13

   struct bucket_lock {
       pthread_mutex_t lock;
   };

   struct bucket_lock bucket_locks[NUM_BUCKETS];
   ```

   - 在 `bcache_init()` 函数中初始化这些锁。

   ```c
   void bcache_init() {
       for (int i = 0; i < NUM_BUCKETS; i++) {
           pthread_mutex_init(&bucket_locks[i].lock, NULL);
       }
       // 其他初始化代码...
   }
   ```

2. **修改 `bget` 和 `brelse` 函数**：
   - 更新 `bget` 函数，使用哈希表来查找适当的桶锁。

   ```c
   struct buf* bget(uint dev, uint blockno) {
       struct buf* b;
       int bucket_index = blockno % NUM_BUCKETS;

       pthread_mutex_lock(&bucket_locks[bucket_index].lock);

       // 其他代码逻辑...
       
       pthread_mutex_unlock(&bucket_locks[bucket_index].lock);
       return b;
   }
   ```

   - 在 `brelse` 函数中，更新对桶锁的使用，确保对缓存块的释放不会引发锁竞争。

   ```c
   void brelse(struct buf* b) {
       int bucket_index = b->blockno % NUM_BUCKETS;

       pthread_mutex_lock(&bucket_locks[bucket_index].lock);

       // 其他代码逻辑...
       
       pthread_mutex_unlock(&bucket_locks[bucket_index].lock);
   }
   ```

3. **重新编译并测试**：
   - 编译并运行程序，检查 `bcachetest` 是否通过，并确认锁竞争显著减少。

   ```bash
   make bcache
   ./bcachetest
   ```

4. **确保代码通过所有测试**：
   - 使用 `make grade` 命令运行所有测试，确保优化后的实现能够通过所有测试用例。

   ```bash
   make grade
   ```

### 3) 实验中遇到的困难和解决办法

1. **处理锁竞争和同步**：
   - **困难描述**：一开始不知道怎么减少缓存块锁的冲突。
   - **解决办法**：后来了解到可以根据数据块的 blocknumber将其保存进一个哈希表， 哈希表的每个 bucket 都有一个相应的锁来保护。

2. **处理哈希冲突**：
   - **困难描述**：然后就是他这个块缓存是由一个包含多个缓存块的链表组成的，将双向链表代码的编写也是比较复杂的， 我当时做的时候这个地方卡了蛮久的。
   - **解决办法**：调整桶的数量和哈希算法，确保哈希冲突尽可能减少。

3. **调试和验证**：
   - **困难描述**：调试多线程程序可能会遇到难以复现的错误。
   - **解决办法**：使用调试工具（如 `gdb`）逐步检查线程状态和锁的实现，确保线程在访问缓冲区块时正确同步。

### 4) 实验心得

通过本次实验，我深入理解了缓冲区缓存管理和锁机制的优化。实现细粒度锁机制让我掌握了如何有效地减少多线程环境中的锁竞争，提高系统的性能和稳定性。通过对哈希表和桶锁的实际应用，我提高了在复杂并发环境中的编程能力，这些技能在高性能系统的开发中非常重要。



