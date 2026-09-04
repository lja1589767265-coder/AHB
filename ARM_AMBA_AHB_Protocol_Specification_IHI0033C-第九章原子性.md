# AMBA AHB Protocol Specification IHI 0033C：第九章 Atomicity 原子性精读

> 原始资料：[AMBA AHB Protocol Specification IHI 0033C](ARM_AMBA_AHB_Protocol_Specification_IHI0033C.pdf)  
> 精读范围：Chapter 9 Atomicity，PDF 第 77～80 页（文档页码 9-77～9-80）  
> 文档版本：Issue C，ID090921  
> 发布日期：2021 年 9 月 15 日  
> 前置阅读：[第八章 Signal validity 信号有效性精读](ARM_AMBA_AHB_Protocol_Specification_IHI0033C-第八章信号有效性.md)

本文面向第一次系统学习 AHB 的读者，统一使用 Issue C 的术语：`Manager`（原 `Master`）、`Subordinate`（原 `Slave`）和 `Interconnect`（互连）。第九章定义两种观察内存更新的原子属性：Single-copy atomicity size 关注一个原子数据块会不会被观察到“撕裂”，Multi-copy atomicity 关注同一次写入会不会只对部分 Agent 可见。

> **颜色说明：** 绿色文字表示可直接用于设计与验证的结论；橙色文字表示容易误读的限制。
>
> **信息类型：** 正文用于忠实转述 Chapter 9 的要求；“理解提示”用于给出观察者视角和数据示例；“工程推断”用于说明实现或验证时如何应用这些属性。

<details>
<summary><strong>1. 原子性讨论的不是总线握手是否完成</strong></summary>

AHB 的 `HREADY` 和 `HRESP` 回答一笔总线传输何时完成、结果是否成功。原子性回答的是另一个层面的问题：当写入已经影响存储状态时，其他观察者可能看到哪些中间结果，以及不同观察者看到写入的顺序和范围是否一致。

Chapter 9 使用两种互相独立的观察角度：

| 属性 | 观察重点 | 要阻止的现象 |
| --- | --- | --- |
| Single-copy atomicity size | 一个观察者读取某个原子数据块时会看到什么 | 同一原子块中一部分字节已更新、另一部分仍是旧值，也就是 tearing（撕裂） |
| Multi-copy atomicity | 多个 Agent 之间如何观察同一地址的写入 | 写入只对部分 Agent 可见，或不同 Agent 以不同顺序观察同一地址的多次写入 |

这里的 Agent 可以理解为能够发起、接收或观察相关内存访问的系统参与者。具体哪些组件属于同一原子组、哪些 Agent 能观察某个地址，由系统结构和属性定义决定。

> **理解提示：** Single-copy 的 “copy” 可以先理解为“从某个观察者看到的一份内存状态”；Multi-copy 则把问题扩大到多个观察者之间的可见性一致性。它们不是在统计系统里实际存在多少份物理存储副本。

> <span style="color:#63d297"><strong>结论：</strong>总线传输成功只说明访问已经被接受；是否无撕裂、是否同时对所有相关 Agent 可见，还要分别检查两类原子性保证。</span>

</details>

<details>
<summary><strong>2. Single-copy atomicity size：一个原子块不能被看见一半</strong></summary>

### 2.1 原子性大小按通信组件组定义

Single-copy atomicity size 定义一笔传输保证以原子方式更新多少个数据字节。这个大小针对**一组互相通信的组件**定义，不一定是整个系统唯一的全局常数。

规范给出的示例包含两个范围不同的组：

- Processor、DSP 和 DRAM Controller 可以构成一个 64-bit Single-copy atomic group，也就是 8 字节原子组；
- 把 DMA、DRAM、SRAM 和外设等更多组件纳入后，更大的系统组可能只保证 32-bit，也就是 4 字节原子性。

这两个保证可以同时存在。同一次访问经过不同组件和观察路径时，能够依赖的原子粒度取决于相关通信组件共同支持的组，而不能只看 Manager 或存储器某一端的数据总线位宽。

> <span style="color:#ffb454"><strong>易错点：</strong>64 位数据总线不自动等于 64-bit Single-copy atomicity；总线宽度说明一次能够传多少数据，原子组属性说明观察者不会看到多大的部分更新。</span>

### 2.2 原子性是一种观察保证

当写传输更新一个存储位置时，对任一观察者都必须保证：要么还没有观察到该位置的更新，要么已经观察到至少一个 Single-copy atomicity size 大小的数据更新。不能先观察到同一原子范围内的一部分字节改变，再在稍后观察到其余字节改变。

假设一个 32-bit 原子元素原来是：

```text
旧值：0xAAAABBBB
新值：0x11223344
```

对于能够依赖 32-bit Single-copy atomicity 的观察者，允许看到：

- 完整旧值 `0xAAAABBBB`；
- 完整新值 `0x11223344`。

不允许先看到高 16 位已经变成 `0x1122`、低 16 位仍是 `0xBBBB`，随后才看到完整新值。也就是说，不能把同一个 32-bit 原子元素的更新拆成两个可被其他 Manager 分别观察到的 16-bit 步骤。

规范特别指出，判断原子性时不考虑数据值在实现内部更新的**精确物理时刻**。存储阵列、缓冲区或总线适配器可以分步骤工作，只要任何 Manager 都无法观察到原子数据的部分更新形式即可。

链表指针是一个典型例子。如果系统把指针作为 32-bit 原子元素，更新指针时就不能让其他 Manager 读到新旧各一半的地址。更复杂的系统可能需要 64-bit 原子元素，以支持基于 64 位值的数据结构。

> <span style="color:#63d297"><strong>结论：</strong>Single-copy atomicity 约束的是对外可观察状态，而不是要求存储电路的所有物理位在完全相同的瞬间翻转。</span>

### 2.3 起始地址对齐限制原子保证

一笔传输得到的 Single-copy atomicity 保证不会大于其起始地址的对齐程度。规范给出的直接例子是：在 64-bit 原子组中，如果 Burst 起始地址没有按 8 字节边界对齐，就不具有 64-bit Single-copy atomicity 保证。

| 原子组目标 | 满足该粒度所需的起始地址条件 | 不满足时可以得出的结论 |
| --- | --- | --- |
| 32-bit（4 字节） | `Start_Address mod 4 = 0` | 不能仅凭该组宣称本次传输具有 32-bit 保证 |
| 64-bit（8 字节） | `Start_Address mod 8 = 0` | 不具有 64-bit 保证 |

例如，地址 `0x1000` 按 8 字节对齐，可以满足 64-bit 保证的对齐前提；地址 `0x1004` 只按 4 字节对齐，即使参与组件属于 64-bit 原子组，也不能对这笔从 `0x1004` 开始的传输宣称 64-bit 原子性。

这里不能继续擅自推导“所以一定还有 32-bit 原子性”。较小粒度是否得到保证，还要看相关组件组是否定义并支持该粒度，以及该起始地址是否满足相应对齐条件。

> <span style="color:#63d297"><strong>结论：</strong>原子组给出可保证的能力上限，起始地址对齐决定当前传输能否实际使用这个上限。</span>

### 2.4 大于原子粒度的传输按原子块更新

如果一笔传输大于 Single-copy atomicity size，内存更新必须以**至少达到该原子大小**的块进行。

以 32-bit 原子组中的 128-bit 更新为例，系统必须保证每个对外可见的更新块至少为 32 bit。Chapter 9 并不要求整个 128-bit 值一次性原子更新，因此观察者可能在某个时刻看到部分 32-bit 块已经是新值，其余块仍是旧值；但不能看到某个 32-bit 块内部只更新了部分字节。

“至少 32 bit”也不等于实现必须恰好每次更新 32 bit。实现可以一次原子更新 64 bit 或完整 128 bit，只要不低于该组保证的最小原子块，并满足起始地址对齐限制。

> **理解提示：** Single-copy atomicity size 是最小不可撕裂粒度，不是所有大传输都必须被切成固定大小的物理写操作。

> <span style="color:#63d297"><strong>结论：</strong>超出原子粒度的大传输可以分块对外可见，但每个可见更新块都不能小于保证的原子大小。</span>

### 2.5 Byte strobe 不会缩小原子性大小

与一笔传输关联的 byte strobe 不影响 Single-copy atomicity size。Sparse write 可以只修改原子块内被 `HWSTRB` 选中的字节，但这不会把系统声明的原子粒度缩小为“置位 strobe 的字节数”。

例如，在满足对齐条件的 32-bit 原子块中，`HWSTRB` 可以只允许 byte 0 和 byte 2 写入。未选中的 byte 1 和 byte 3 保持原值；对观察者而言，允许看到写入前的完整 32-bit 状态或 sparse write 完成后的完整 32-bit 状态，不能先看到 byte 0 已更新、随后才看到 byte 2 更新。

这里的“32-bit 原子更新”不表示四个字节的数值都必须变化，而表示这个 4 字节范围内由本次写入造成的状态变化不能以更小的可观察步骤暴露出去。

> <span style="color:#63d297"><strong>结论：</strong><code>HWSTRB</code> 决定哪些字节真正改变，Single-copy atomicity size 决定这些改变以多大的不可撕裂范围对外可见。</span>

</details>

<details>
<summary><strong>3. Multi-copy atomicity：不能只让部分 Agent 先看见</strong></summary>

AHB5 定义了系统属性 `Multi_Copy_Atomicity`：

- `Multi_Copy_Atomicity=True`：系统声明提供 Multi-copy atomicity；
- 不支持或没有声明该属性时，默认值为 `False`。

默认值为 `False` 的含义是**不能依赖这项系统级保证**，并不表示系统在每次运行中都必然出现顺序分歧或部分可见。

### 3.1 同一地址的写入顺序对所有 Agent 一致

假设 Manager A 和 Manager B 分别向同一地址 X 写入 `W1` 和 `W2`，而这两个竞争写入的全局先后尚未预先确定。在 Multi-copy atomic 系统中，无论系统最终呈现哪一种顺序，所有相关 Agent 对这两个写入的观察顺序都必须一致。

| Agent C 的观察顺序 | Agent D 的观察顺序 | 是否满足要求 |
| --- | --- | --- |
| `W1 -> W2` | `W1 -> W2` | 是 |
| `W2 -> W1` | `W2 -> W1` | 是，前提是系统对所有 Agent 都呈现这一顺序 |
| `W1 -> W2` | `W2 -> W1` | 否 |

本章并没有用这条规则指定所有不同地址写入之间的完整全局顺序。它明确约束的是**写到同一位置**的操作，不能被不同 Agent 观察成互相矛盾的先后关系。

> <span style="color:#63d297"><strong>结论：</strong>Multi-copy atomicity 要求所有相关 Agent 对同一地址写入形成一致的观察顺序。</span>

### 3.2 一旦对非发起者可见，就必须对所有 Agent 可见

第二条规则关注写入的传播范围：如果发起者之外的某个 Agent 已经能够观察到对地址 X 的写入，那么该写入也必须对所有相关 Agent 可观察。

假设 Manager A 发出写入，Agent B 和 Agent C 都能访问 X：

- A 仍可在自己的私有缓冲路径中暂时看到某种本地状态；
- 但只要 B 作为非发起者已经能观察到新值，就不能仍让 C 只能观察旧值；
- 这里约束的是架构上的可观察性，不要求信号在所有物理端口上以模拟意义的同一瞬间改变。

> <span style="color:#63d297"><strong>结论：</strong>写入不能进入“已经对 B 公开、却仍未对 C 公开”的系统可见状态。</span>

### 3.3 Forwarding buffer 为什么可能破坏这项属性

Forwarding buffer 可能把尚未全局可见的写入提前转发给某个读取者。例如：

1. Manager A 的写入 W 暂存在缓冲区中；
2. Agent B 的读取命中转发路径，因此已经得到 W 的新值；
3. Agent C 通过另一条路径访问同一地址，仍得到旧值；
4. 同一写入此时只对部分 Agent 可见，不满足 Multi-copy atomicity。

规范指出，避免使用这种会使传输只对部分 Agent 可见的 forwarding buffer，可以确保 Multi-copy atomicity。这不是说所有缓冲结构都被禁止；如果缓冲和一致性机制能够保证写入的全局可见性规则，仍可采用相应实现。

对于包含硬件缓存一致性的系统，保证 Multi-copy atomicity 还需要额外要求。IHI 0033C 明确说明这些要求存在，但没有在本规范中继续展开，因此不能只凭 Chapter 9 的两条文字规则完成复杂一致性系统的全部设计证明。

> <span style="color:#ffb454"><strong>易错点：</strong>“有 Cache Coherency”不能自动替代 <code>Multi_Copy_Atomicity=True</code> 的系统声明和验证；具体一致性协议仍需满足额外要求。</span>

</details>

<details>
<summary><strong>4. 两类原子性与锁定、Exclusive 的区别</strong></summary>

### 4.1 Single-copy 与 Multi-copy 相互不能替代

| 维度 | Single-copy atomicity | Multi-copy atomicity |
| --- | --- | --- |
| 核心问题 | 一个原子块是否被撕裂 | 一个写入是否按一致顺序对全部 Agent 可见 |
| 典型失败 | 32-bit 值被观察为新旧各 16 bit | B 已看到新值而 C 仍只看到旧值 |
| 作用范围 | 定义过的通信组件组和原子大小 | 声明 `Multi_Copy_Atomicity=True` 的系统 |
| 关键限制 | 起始地址对齐、最小更新块 | 同地址顺序一致、非发起者可见后全局可见 |

一个系统可以保证每个 32-bit 写入都不撕裂，却仍通过不同传播路径让 B 比 C 更早观察到写入。反过来，多 Agent 的传播顺序规则也不能代替对单个宽数据元素是否撕裂的粒度说明。因此，验证其中一项不能作为另一项已经成立的证明。

### 4.2 原子性、Locked transfer 和 Exclusive Transfer 解决不同问题

- **Single-copy atomicity：** 规定内存状态以多大粒度不可撕裂地被观察。
- **Multi-copy atomicity：** 规定写入在多个 Agent 之间的观察顺序和可见范围。
- **Locked transfer：** 用 `HMASTLOCK` 表示一段传输序列不可分割地处理，限制其他访问插入该序列。
- **Exclusive Transfer：** 通过 Exclusive Read、Exclusive Write 和监视机制判断某个条件写入能否成功，详见 Chapter 10。

`HMASTLOCK` 或 `HEXCL` 不是 Chapter 9 两个属性的替代开关；同样，声明某个原子大小也不会自动产生 read-modify-write 语义。软件或硬件如果需要“读取旧值、计算、条件写回”这样的复合操作，还要使用适当的同步原语和下一章定义的机制。

> <span style="color:#63d297"><strong>结论：</strong>Chapter 9 定义可观察内存状态的保证；Locked 和 Exclusive 则约束访问序列或条件更新，四者不能混为同一概念。</span>

</details>

<details>
<summary><strong>5. 设计与验证检查清单</strong></summary>

### 系统与实现侧

- 明确定义每个 Single-copy atomic group 包含哪些 Manager、存储器、控制器、Interconnect 和外设。
- 为每个组记录原子性大小，不能根据数据总线宽度或一次传输大小自行猜测。
- 检查端到端路径上的宽窄转换、桥接、缓冲和存储写入机制不会把更新暴露成小于保证粒度的步骤。
- 在使用原子保证前检查起始地址对齐；64-bit 保证要求 8 字节对齐，32-bit 保证要求 4 字节对齐。
- 大于原子粒度的写入允许分块时，确保每个对外可见的更新块至少达到声明粒度。
- Sparse write 或 byte strobe 不得被实现成可观察的逐字节分阶段更新。
- 只有经过系统级论证后才声明 `Multi_Copy_Atomicity=True`；未声明时按默认 `False` 处理。
- 审查 forwarding buffer、旁路、Cache 和一致性节点，确认写入不会只对部分 Agent 提前可见。

### 验证侧

- Single-copy 检查应放在可观察内存状态或多个 Manager 的访问结果上，不能只检查一笔 AHB 握手是否成功。
- 使用旧值和新值差异明显的测试数据，主动捕获由不同字节拼成的 torn value。
- 覆盖对齐和非对齐起始地址，区分“属性允许保证”和“本次传输实际满足保证”。
- 覆盖小于、等于和大于原子性大小的传输，并对大传输检查最小可见更新块。
- 覆盖不同 `HWSTRB` 组合，尤其是非连续 sparse write，确认 strobe 不会缩小不可撕裂范围。
- Multi-copy 验证至少需要两个非发起 Agent，从不同缓冲或一致性路径观察同一地址。
- 让多个 Manager 对同一位置竞争写入，比较所有 Agent 记录到的写入顺序是否一致。
- 对 `Multi_Copy_Atomicity=False` 的系统，不应把一次测试中“没有观察到违规”误写成系统已经提供该保证。
- 含硬件 Cache Coherency 的系统还要依据相应一致性协议和内存模型补充验证，Chapter 9 本身不足以覆盖全部要求。

</details>

<details>
<summary><strong>6. 易错点与记忆方法</strong></summary>

### 七个易错点

1. **把传输成功等同于原子更新。** `HREADY=1`、`HRESP=0` 不能单独证明无撕裂或全局可见。
2. **把总线宽度当成原子性大小。** 两者属于不同的系统属性。
3. **忽略通信组件组。** 局部 64-bit 原子组不代表包含所有外设的更大组也有 64-bit 保证。
4. **忽略起始地址对齐。** 64-bit 原子组中的非 8 字节对齐 Burst 没有 64-bit 保证。
5. **认为 byte strobe 会缩小原子粒度。** `HWSTRB` 只决定实际修改的字节。
6. **把 Single-copy 当成 Multi-copy。** 不撕裂不代表不会只对部分 Agent 提前可见。
7. **把默认 `False` 理解成一定发生违规。** 它只表示系统没有提供可依赖的 Multi-copy 保证。

### 一句话记忆

> Single-copy 管“一块数据不能看见一半”，Multi-copy 管“一次写入不能只让一部分人看见”。

</details>

<details>
<summary><strong>7. 自测题</strong></summary>

<details>
<summary>1. Single-copy atomicity size 定义了什么？</summary>

> 它定义一笔传输保证以原子方式更新的数据字节数，也就是相关观察者不能看到小于该粒度的部分更新。

</details>

<details>
<summary>2. 为什么原子性大小要针对通信组件组定义？</summary>

> 因为不同数据路径和参与组件可能支持不同的不可撕裂粒度。局部组件组可以保证 64 bit，而包含更多 Manager、存储器和外设的更大组可能只共同保证 32 bit。

</details>

<details>
<summary>3. 64-bit 原子组中的传输是否一定具有 64-bit 原子性？</summary>

> 不一定。传输起始地址还必须满足 8 字节对齐；如果不对齐，就没有 64-bit Single-copy atomicity 保证。

</details>

<details>
<summary>4. 一笔 128-bit 写入位于 32-bit 原子组时，是否必须整体一次更新？</summary>

> 不必须。它可以分块对外可见，但每个可见更新块必须至少达到 32 bit，并满足相应对齐要求；任何 32-bit 原子块内部不能被观察到部分更新。

</details>

<details>
<summary>5. <code>HWSTRB</code> 只选中一个字节时，Single-copy atomicity size 是否缩小为 1 字节？</summary>

> 不会。Byte strobe 决定哪些字节的值改变，不改变组声明的 Single-copy atomicity size。实现仍不能把本次写入的效果作为小于保证粒度的分阶段状态暴露给观察者。

</details>

<details>
<summary>6. 原子更新是否要求存储器所有物理位在完全同一瞬间翻转？</summary>

> 不要求。规范关注是否有 Manager 能观察到部分更新；内部可以分步骤实现，只要中间状态不可被观察。

</details>

<details>
<summary>7. <code>Multi_Copy_Atomicity=True</code> 要满足哪两项核心规则？</summary>

> 所有 Agent 必须以相同顺序观察同一位置的写入；一笔写入一旦对发起者之外的某个 Agent 可观察，就必须对所有相关 Agent 可观察。

</details>

<details>
<summary>8. 为什么 forwarding buffer 可能破坏 Multi-copy atomicity？</summary>

> 因为它可能把尚未全局可见的写入提前转发给某个 Agent，使该 Agent 看到新值时，其他 Agent 仍只能看到旧值。

</details>

<details>
<summary>9. <code>Multi_Copy_Atomicity=False</code> 是否表示系统一定会出现不一致观察？</summary>

> 不是。它表示系统没有声明可依赖的 Multi-copy atomicity 保证；某次执行没有出现分歧也不能把默认值推断为 `True`。

</details>

<details>
<summary>10. 为什么 Single-copy atomicity 不能替代 Exclusive Transfer？</summary>

> Single-copy atomicity 只保证数据块更新不被撕裂，不负责判断从读取到写回之间是否被其他 Agent 修改。需要条件写入时，还要使用 Exclusive Access Monitor 等下一章定义的机制。

</details>

</details>

## 8. 问题记录与解答

暂无。

<details>
<summary><strong>9. 本篇边界与资料来源</strong></summary>

本篇只解释 Chapter 9 定义的 Single-copy atomicity size 和 Multi-copy atomicity，不展开完整内存一致性模型、Cache Coherency 协议、软件内存屏障或 Exclusive Access Monitor 的实现。相关内容可继续阅读：

- [第三章 Transfers 传输精读](ARM_AMBA_AHB_Protocol_Specification_IHI0033C-第三章传输.md)：传输大小、Burst、等待和完成条件；
- [第六章 Data Buses 数据总线精读](ARM_AMBA_AHB_Protocol_Specification_IHI0033C-第六章数据总线.md)：数据总线宽度和 byte lane；
- [第八章 Signal validity 信号有效性精读](ARM_AMBA_AHB_Protocol_Specification_IHI0033C-第八章信号有效性.md)：`HWDATA`、`HWSTRB` 和返回数据的有效窗口；
- Chapter 10 Exclusive Transfers：Exclusive Read、Exclusive Write、`HEXCL`、`HMASTER`、`HEXOKAY` 和 Exclusive Access Monitor。

本文以 [AMBA AHB Protocol Specification IHI 0033C](ARM_AMBA_AHB_Protocol_Specification_IHI0033C.pdf) 的 Chapter 9（PDF 第 77～80 页，文档页码 9-77～9-80）为资料。Chapter 9 的有效正文位于文档页码 9-78～9-79，第 9-80 页只有续页页眉。原章没有图表，因此本文使用观察者场景、数据示例和条件表格解释原文，没有额外制作装饰性图片。

本文是学习整理，不替代官方规范。实现或验证 AHB 兼容设计时，应以原始 PDF 及系统采用的完整内存模型与一致性规范为准。

</details>
