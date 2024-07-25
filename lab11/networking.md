# Lab11: networking   (网络)

## networking

### 1. 实验目的

在本实验中，你将为 xv6 操作系统开发一个网络接口卡 (NIC) 驱动程序，具体使用 Intel E1000 网络控制器。实验的目标是实现 `e1000_transmit` 和 `e1000_recv` 函数，以完成数据包的发送和接收功能。实验重点是使驱动程序能够通过 E1000 处理网络通信，并与 xv6 网络栈兼容。

### 2. 实验步骤

#### 1. 配置环境

1. **获取源码并切换分支**：
   ```bash
   git fetch
   git checkout net
   make clean
   ```

2. **理解设备手册**：
   - 阅读 Intel E1000 软件开发手册，特别是第2章（设备概述）、第3章（数据包接收和发送）和第13章（寄存器），以便了解设备的工作原理和寄存器定义。

#### 2. 实现 `e1000_transmit` 函数

1. **获取 TX 环形缓冲区索引**：
   - 读取 `E1000_TDT` 控制寄存器，找到下一个预期的传输描述符索引。

   ```c
   uint32_t tx_index = regs[E1000_TDT];
   ```

2. **检查描述符状态**：
   - 如果 `E1000_TXD_STAT_DD` 位未设置，说明上一个传输请求尚未完成，返回错误。

   ```c
   if (!(tx_desc[tx_index].status & E1000_TXD_STAT_DD)) {
       return -1; // 传输描述符未准备好
   }
   ```

3. **释放上一个 mbuf**：
   - 如果描述符准备好，使用 `mbuffree` 释放上一个传输的 mbuf（如果有）。

   ```c
   if (last_mbuf) {
       mbuffree(last_mbuf);
   }
   ```

4. **填充描述符**：
   - 将 mbuf 数据、长度和必要的命令标志设置到 TX 描述符中，并保存 mbuf 指针以便后续释放。

   ```c
   tx_desc[tx_index].addr = (uint64_t)m->head;
   tx_desc[tx_index].length = m->len;
   tx_desc[tx_index].cmd = E1000_TXD_CMD_EOP | E1000_TXD_CMD_IFCS;
   last_mbuf = m;
   ```

5. **更新 TX 环形缓冲区**：
   - 增加 TX 描述符索引并更新 `E1000_TDT` 寄存器。

   ```c
   regs[E1000_TDT] = (tx_index + 1) % TX_RING_SIZE;
   ```

#### 3. 实现 `e1000_recv` 函数

1. **获取 RX 环形缓冲区索引**：
   - 读取 `E1000_RDT` 控制寄存器，找到下一个可用的接收描述符索引。

   ```c
   uint32_t rx_index = (regs[E1000_RDT] + 1) % RX_RING_SIZE;
   ```

2. **检查描述符状态**：
   - 检查 `E1000_RXD_STAT_DD` 位以确定是否有新数据包可用。如果没有，停止处理。

   ```c
   if (!(rx_desc[rx_index].status & E1000_RXD_STAT_DD)) {
       return;
   }
   ```

3. **处理数据包**：
   - 更新 mbuf 的长度，并通过 `net_rx` 函数将 mbuf 传递给网络栈。

   ```c
   m->len = rx_desc[rx_index].length;
   net_rx(m);
   ```

4. **分配新 mbuf**：
   - 使用 `mbufalloc` 分配一个新的 mbuf，并将其数据指针设置到 RX 描述符中。

   ```c
   m = mbufalloc();
   rx_desc[rx_index].addr = (uint64_t)m->head;
   rx_desc[rx_index].status = 0;
   ```

5. **更新 RX 环形缓冲区**：
   - 更新 `E1000_RDT` 寄存器。

   ```c
   regs[E1000_RDT] = rx_index;
   ```

#### 4. 测试

1. **编写并运行测试程序**：
   - 运行 `make server` 和 `make qemu`，然后在 xv6 中运行 `nettests` 来验证实现。

   ```bash
   make server
   make qemu
   ./nettests
   ```

2. **检查结果**：
   - 使用 `tcpdump` 查看 `packets.pcap` 文件中的数据包，确认数据包的发送和接收。

   ```bash
   tcpdump -XXnr packets.pcap
   ```

### 3. 实验中遇到的困难和解决办法

1. **数据包处理的复杂性**：
   - **困难描述**：处理数据包的发送和接收需要精确操作 E1000 寄存器和描述符。
   - **解决办法**：参考 E1000 软件开发手册，仔细检查每个寄存器的使用，并逐步调试代码。

2. **同步问题**：
   - **困难描述**：确保在多线程环境中正确同步访问 E1000 硬件。
   - **解决办法**：使用锁机制保护对 E1000 描述符和寄存器的访问。

### 4. 实验心得

通过本次实验，我深入了解了网络接口卡驱动程序的开发过程，包括数据包的发送和接收、E1000 硬件寄存器的操作以及如何在 xv6 中处理网络通信。实验增强了我对网络设备驱动程序的理解，提高了处理硬件设备和网络协议的能力。这些技能对操作系统的网络功能开发具有重要意义。