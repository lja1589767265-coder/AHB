# AHB 在 APB 的基础上增加了什么？

## 用户原始问题

> **AHB 在 APB 的基础上增加了什么？**

---

# 回答

严格来说，并不是 **AHB 在 APB 的基础上增加了什么**，而是 **AHB（Advanced High-performance Bus）** 和 **APB（Advanced Peripheral Bus）** 是为不同应用场景设计的两种总线。

- **AHB**：高性能系统总线，面向 CPU、DMA、SRAM 等高速模块。
- **APB**：低功耗外设总线，面向 UART、SPI、GPIO、Timer 等低速外设。

如果把 APB 看作“基础版”，那么 AHB 可以理解为增加了许多为了提高性能而设计的特性。

---

# 总体对比

| 功能 | APB | AHB 增加的能力 |
|------|------|--------------|
| 流水线（Pipeline） | ❌ | ✅ 地址阶段与数据阶段重叠，提高吞吐率 |
| Burst 连续传输 | ❌ | ✅ 支持 INCR、WRAP 等 Burst |
| 多 Master | ❌ | ✅ CPU、DMA 等多个 Master |
| 总线仲裁（Arbiter） | ❌ | ✅ 多个 Master 竞争总线 |
| 数据总线宽度 | 一般 32 bit | 可支持 64/128 bit 甚至更宽 |
| Master 快速切换 | ❌ | ✅ 单周期切换 |
| 等待/错误处理 | PREADY | HREADY、ERROR、RETRY、SPLIT |
| 高速 Memory 访问 | 不适合 | ✅ 专门优化 |

---

# 1. 流水线（Pipeline）

这是 AHB 和 APB 最大的区别。

## APB

```text
Cycle1   地址
Cycle2   数据
Cycle3   地址
Cycle4   数据
```

每次必须等待当前数据传输结束，才能开始下一次访问。

---

## AHB

```text
Cycle1    Addr1
Cycle2    Addr2   Data1
Cycle3    Addr3   Data2
Cycle4    Addr4   Data3
```

地址阶段和数据阶段可以重叠。

因此：

- APB：一次只能完成一笔传输。
- AHB：可以一边传输上一笔数据，一边准备下一笔地址。

这样可以显著提高总线吞吐率。

---

# 2. Burst 连续传输

## APB

```text
CPU
 ├── 读 0x1000
 ├── 读 0x1004
 └── 读 0x1008
```

每一次访问都是独立的 Transaction。

---

## AHB

```text
Start Address = 0x1000

Burst = INCR4

Data:
0x1000
0x1004
0x1008
0x100C
```

地址只发送一次。

随后 Memory 连续输出数据。

这也是 DDR、SRAM 等高速存储器通常使用 AHB 的原因之一。

---

# 3. 多 Master

## APB

```text
CPU
 |
APB
 |
UART
```

APB 只有一个 Master。

---

## AHB

```text
        CPU
         |
DMA -----|
         |
USB -----|
         |
      Arbiter
         |
        AHB
```

AHB 可以支持：

- CPU
- DMA
- USB
- GPU（部分 SoC）

多个 Master 同时申请总线。

因此必须增加仲裁器（Arbiter）。

---

# 4. 总线仲裁（Arbiter）

例如：

```text
CPU 请求

DMA 请求

USB 请求
```

Arbiter 判断：

```text
Grant CPU
```

下一拍：

```text
Grant DMA
```

因此：

AHB 比 APB 多了一套完整的仲裁机制。

---

# 5. 更宽的数据总线

APB：

```text
32 bit
```

AHB：

```text
32 bit
64 bit
128 bit
```

总线越宽：

一次传输的数据越多。

更适合高速 Memory。

---

# 6. 更完善的等待机制

APB：

只有

```text
PREADY
```

Slave 没准备好：

```text
PREADY = 0
```

Master 只能一直等待。

---

AHB：

除了

```text
HREADY
```

还有：

```text
ERROR

RETRY

SPLIT（AHB2）
```

Slave 可以通知 Master：

- 出错
- 稍后重试
- 暂时释放总线

因此总线利用率更高。

---

# 7. Master 快速切换

例如：

```text
Cycle5

CPU 完成
```

下一拍：

```text
Cycle6

DMA Addr
```

DMA 可以立即开始。

不会浪费 Bus 周期。

---

# 8. AHB-APB Bridge

一个典型 SoC 的结构如下：

```text
                CPU
                 |
             AHB Bus
        _________|_________
       |         |         |
     SRAM      DMA      Bridge
                         |
                      APB Bus
               _________|________
              |        |        |
            UART    SPI     TIMER
```

高速模块：

- CPU
- SRAM
- DMA

全部连接 AHB。

低速模块：

- UART
- GPIO
- Timer
- SPI
- Watchdog

全部连接 APB。

中间通过 **AHB-APB Bridge** 完成协议转换。

---

# 总结

如果把 APB 看作基础版，那么 **AHB 主要增加了以下高性能特性：**

- ✅ 流水线（Pipeline）
- ✅ Burst 连续传输
- ✅ 多 Master
- ✅ 总线仲裁（Arbiter）
- ✅ 更宽的数据总线
- ✅ Master 快速切换
- ✅ 更完善的等待/错误处理机制（HREADY、ERROR、RETRY、SPLIT）
- ✅ 更适合高速 Memory 和 DMA 访问

---

# 一句话总结

> **APB 追求的是“简单、低功耗”；AHB 追求的是“高吞吐、高带宽、高性能”。**

因此，在绝大多数 SoC 中：

- **AHB** 作为系统主干总线，连接 CPU、DMA、Memory 等高速模块；
- **APB** 作为低速外设总线，连接 UART、GPIO、SPI、Timer 等外设；
- 两者之间通过 **AHB-APB Bridge** 进行协议转换。