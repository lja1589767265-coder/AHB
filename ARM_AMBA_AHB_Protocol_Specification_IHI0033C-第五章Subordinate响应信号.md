# AMBA AHB Protocol Specification IHI 0033C：第五章 Subordinate Response Signaling 响应信号精读

> 原始资料：[ARM_AMBA_AHB_Protocol_Specification_IHI0033C.pdf](ARM_AMBA_AHB_Protocol_Specification_IHI0033C.pdf)<br>
> 精读范围：Chapter 5 Subordinate Response Signaling，PDF 第 59-62 页（文档页码 5-59～5-62）<br>
> 文档版本：Issue C，ID090921，2021 年 9 月 15 日发布<br>
> 前置阅读：[第四章 Bus Interconnection 总线互连精读](ARM_AMBA_AHB_Protocol_Specification_IHI0033C-第四章总线互连.md)

本文面向第一次系统学习 AHB 的读者，统一使用 Issue C 的术语 `Manager（原 Master）`、`Subordinate（原 Slave）` 和 `Interconnect（互连）`。本章重点解释：Subordinate 如何用 `HRESP` 与 `HREADYOUT` 共同表示传输仍在等待、成功完成或错误完成，以及两周期 `ERROR` 为什么是 AHB 流水线所必需的。

> **颜色约定：** <span style="color:#4ea1ff">蓝色</span>表示关键协议名和信号，<span style="color:#ffb454">橙色</span>表示限制和易错条件，<span style="color:#63d297">绿色</span>表示正确方向和结论。
>
> **信息标记：** 正文默认是对 Chapter 5 规范原意的中文转述；“理解提示”用于补充流水线直观解释；“工程推断”用于说明实现或验证层面的结论。涉及 Interconnect、数据有效性和 Exclusive Transfers 时，会明确指出对应的其他章节。

## 目录

- 1. 响应的两个维度：结果与完成状态
- 2. 成功完成与普通等待
- 3. 两周期 ERROR 响应
- 4. Figure 5-1 逐拍解析
- 5. ERROR 之后的 Burst 与读数据
- 6. 实现与验证检查点
- 7. 易错点、口诀与自测
- 8. 问题记录与解答
- 9. 本章边界与资料来源

<a id="1-响应的两个维度结果与完成状态"></a>
<details>
<summary><strong>1. 响应的两个维度：结果与完成状态</strong></summary>

Subordinate 被访问时，必须返回当前传输的状态。完整响应由两个维度共同组成（IHI 0033C，5.1 节，第 5-60 页）：

- `HRESP` 回答“传输结果是什么”；
- `HREADYOUT` 回答“当前数据阶段是否能够完成”。

因此，不能脱离完成状态单独解释 `HRESP`，也不能脱离传输结果单独解释 `HREADYOUT`。在系统级接口上，Interconnect 会把目标 Subordinate 的局部返回选择为 Manager 看到的 `HRESP` 和 `HREADY`。

### 1.1 `HRESP` 只说明结果类型

Issue C 中 `HRESP` 是 1 bit 信号，只有两种编码：

| `HRESP` | 响应 | 规范含义 |
| --- | --- | --- |
| `0` | `OKAY` | 当前传输可能已经成功完成，也可能仍需更多周期；必须再看完成状态 |
| `1` | `ERROR` | 当前传输发生错误；错误必须通知 Manager，并使用规定的两周期响应 |

这是对原规范 Table 5-1 的中文整理。表中最重要的信息是：<span style="color:#ffb454"><strong>`HRESP=0` 不能单独证明传输已经成功完成</strong></span>。等待周期也必须输出 `OKAY`，所以还要结合 `HREADYOUT` 或系统级 `HREADY`。

> **版本边界：** IHI 0033C Issue C 描述 AHB-Lite 与 AHB5，`HRESP` 为 1 bit，仅编码 `OKAY` 和 `ERROR`。旧版完整 AHB 中的 `RETRY`、`SPLIT` 及 2 bit 响应不能混入本文的状态表。

### 1.2 `HREADYOUT` 说明局部传输是否完成

从目标 Subordinate 的本地接口观察：

- `HREADYOUT=0`：Subordinate 还不能结束当前数据阶段；
- `HREADYOUT=1`：Subordinate 允许当前数据阶段结束。

将 `HRESP` 和 `HREADYOUT` 组合后，才得到完整响应：

| `HRESP` | `HREADYOUT=0` | `HREADYOUT=1` |
| --- | --- | --- |
| `0`（`OKAY`） | 传输等待中 | 传输成功完成 |
| `1`（`ERROR`） | `ERROR` 第一个周期 | `ERROR` 第二个周期，错误完成 |

这是对原规范 Table 5-2 的中文整理。四个格子都具有明确含义，不能只把 `HREADYOUT` 当作“成功位”，也不能只把 `HRESP` 当作“完成位”。

### 1.3 同一组响应，接口两侧的信号名不同

Chapter 5 的响应表从 Subordinate 本地输出出发，所以使用 `HREADYOUT`。在包含多个 Subordinate 的系统中，Interconnect 选择当前数据阶段对应的返回信号，Manager 看到的完成信号就叫 `HREADY`。

| 要回答的问题 | Subordinate 侧输出 | Manager 侧看到 |
| --- | --- | --- |
| 当前传输完成了吗？ | `HREADYOUT_x` | `HREADY` |
| 当前传输结果是什么？ | `HRESP_x` | `HRESP` |

<span style="color:#63d297"><strong>可以把这条关系记成：<code>HREADYOUT_x</code> 经 Interconnect 选择后形成 <code>HREADY</code>；<code>HRESP_x</code> 经同一选择后形成 <code>HRESP</code>。</strong></span>

也就是说，Interconnect 不是重新定义响应含义，而是把目标 Subordinate 的局部返回送到系统接口。Manager 最终仍要同时看完成状态和结果状态。

因此，对 Manager 而言：

- `HREADY=0, HRESP=0`：当前传输仍在普通等待；
- `HREADY=1, HRESP=0`：当前传输成功完成；
- `HREADY=0, HRESP=1`：两周期 `ERROR` 的第一个周期；
- `HREADY=1, HRESP=1`：两周期 `ERROR` 的第二个周期，当前传输错误完成。

> **理解提示：** 本章后面解析 Figure 5-1 时使用图中的系统级 `HREADY`。设计或验证时应先确定观察的是 Subordinate 本地接口还是 Manager 系统接口，不能把两层信号名混在同一条断言中。

</details>

<a id="2-成功完成与普通等待"></a>
<details>
<summary><strong>2. 成功完成与普通等待</strong></summary>

### 2.1 立即成功完成

Subordinate 如果能够立即完成请求，就在当前数据阶段给出：

```text
HREADYOUT=1
HRESP=0       // OKAY
```

这表示零等待成功。经过 Interconnect 选择后，Manager 看到 `HREADY=1, HRESP=0`，在该边沿结束当前传输。

### 2.2 先等待，再成功完成

如果 Subordinate 已经接受一笔传输，但还不能在当前周期完成，就先把 `HREADYOUT` 置为 `0`，让数据阶段继续等待。<span style="color:#ffb454"><strong>这里的 `HRESP=0` 只表示“没有报告错误”，不表示传输已经成功。</strong></span>

这条“最终成功”的路径可以按两个阶段理解：

| 数据阶段 | `HREADYOUT` | `HRESP` | Manager 侧对应状态 | 含义 |
| --- | --- | --- | --- | --- |
| 等待周期（一个或多个） | `0` | `0`（`OKAY`） | `HREADY=0, HRESP=0` | 当前传输还没完成，继续等待 |
| 最后一个周期 | `1` | `0`（`OKAY`） | `HREADY=1, HRESP=0` | 当前传输成功完成 |

例如，Subordinate 需要两个等待周期时，响应过程是：

```text
第 1 个周期：HREADYOUT=0, HRESP=0  // 等待
第 2 个周期：HREADYOUT=0, HRESP=0  // 继续等待
第 3 个周期：HREADYOUT=1, HRESP=0  // 成功完成
```

如果不需要等待，就直接进入最后一行，也就是第 2.1 节所说的零等待成功。

> <span style="color:#63d297"><strong>记住：成功路径中 `HRESP` 始终为 `0`；真正区分“等待”和“完成”的是 `HREADYOUT` 从 `0` 变为 `1` 的那个周期。</strong></span>

如果 Subordinate 最终判定传输失败，就不再沿用这条成功路径，而要进入第 3 节的两周期 `ERROR` 响应。

### 2.3 等待会阻塞整个接口

规范注记指出，通常每个 Subordinate 都应预先确定自己最多插入多少个等待周期，以便系统计算最坏访问延迟（IHI 0033C，第 5-61 页）。

AHB 的等待会停住当前数据阶段，并连带阻止流水线中的下一地址阶段继续推进。因此，长时间保持 `HREADYOUT=0` 不只是让当前 Subordinate 变慢，也会降低整个 AHB 接口的吞吐率。

> **工程推断：** 对具有可变延迟的 Subordinate，可以在接口规格中明确最大等待周期，并在验证环境设置超时检查。若内部操作可能无限等待，应由系统设计额外定义超时或错误恢复机制；Chapter 5 本身不规定统一的超时数值。

</details>

<a id="3-两周期-error-响应"></a>
<details>
<summary><strong>3. 两周期 <code>ERROR</code> 响应</strong></summary>

Subordinate 发现当前传输出错时（例如向只读地址写入），必须返回两周期 `ERROR` 响应，而不能在一个周期内结束。下面先看两周期的信号组合，再解释它为什么需要两拍。

### 3.1 `ERROR` 必须占用两个周期

与单周期即可完成的 `OKAY` 不同，`ERROR` 必须按以下两拍输出：

| 阶段 | `HRESP` | `HREADYOUT` | 含义 |
| --- | --- | --- | --- |
| `ERROR` 第一个周期 | `1` | `0` | 已报告错误，但故意再延长当前传输一个周期 |
| `ERROR` 第二个周期 | `1` | `1` | 保持错误响应并结束当前传输 |

因此，错误响应的固定结尾是：

```text
(1,0) -> (1,1)
```

<span style="color:#ffb454"><strong>不能用孤立的 `(1,1)` 在一个周期内立即结束错误传输</strong></span>，也不能把 `(1,0)` 当作可以任意重复的普通等待状态。

### 3.2 错误前可以有额外等待

如果 Subordinate 需要更多时间才能决定是否报错，可以先插入普通等待周期。此时必须保持 `HRESP=0`，直到正式进入两周期 `ERROR`：

```text
(0,0)* -> (1,0) -> (1,1)
```

也就是说，增加错误判定延迟的方法是把 `(0,0)` 放在错误序列之前，而不是延长或重复 `ERROR` 本身的两个周期。

### 3.3 为什么必须给 Manager 两个周期

AHB 的地址阶段和数据阶段流水线重叠。Subordinate 开始返回当前传输 A 的 `ERROR` 时，下一笔传输 B 的地址通常已经广播到总线上。

两周期响应提供了以下时间窗口：

1. 第一个 `ERROR` 周期令 `HREADY=0`，阻止流水线继续推进，并明确告诉 Manager 当前传输将失败。
2. Manager 可以把下一笔访问意图的 `HTRANS` 改为 `IDLE`，撤销这笔跟随访问。
3. 第二个 `ERROR` 周期令 `HREADY=1`，以 `HRESP=1` 正式结束当前传输。

> <span style="color:#63d297"><strong>关键区别：</strong></span> 两周期 `ERROR` 完成的是当前传输 A，同时为流水线中的下一笔访问 B 提供取消窗口；它不允许 Manager 取消已经开始的 A。

</details>

<a id="4-figure-5-1-逐拍解析"></a>
<details>
<summary><strong>4. Figure 5-1 逐拍解析</strong></summary>

![Figure 5-1 展示带一个前置等待周期的两周期 ERROR 响应](assets/ihi0033c-chapter5/figure-5-1-error-response.png)

*图 1：原规范 Figure 5-1。传输 A 先经历一个普通等待周期，再经历两周期 `ERROR`；Manager 随后把下一笔地址 B 对应的 `HTRANS` 改为 `IDLE`。图中的 `HREADY` 是系统级完成状态，`HADDR[31:0]` 和 `HWDATA[31:0]` 的 32 位宽度仅为本图示例。来源：IHI 0033C，第 5-61 页。Copyright © 2001, 2006, 2010, 2015, 2021 Arm Limited or its affiliates. All rights reserved.*

图中 A 是发生错误的写传输，B 是流水线中已经出现的下一笔访问意图。逐拍关系如下：

| 时段 | `HREADY` | `HRESP` | 当前数据阶段 | 地址阶段及 Manager 动作 |
| --- | --- | --- | --- | --- |
| T0～T1 | `1` | `OKAY` | 前一状态完成 | A 的 `NONSEQ` 地址阶段出现 |
| T1～T2 | `0` | `OKAY` | A 插入一个普通等待周期，`Data(A)` 保持 | B 的地址已出现在流水线上，但不能据此认为 A 已结束 |
| T2～T3 | `0` | `ERROR` | A 的 `ERROR` 第一个周期 | 规范说明 B 在 T2 被 Subordinate 登记；总线被继续停顿 |
| T3～T4 | `1` | `ERROR` | A 的 `ERROR` 第二个周期并错误完成 | Manager 将 `HTRANS` 改为 `IDLE`，取消原本指向 B 的访问意图 |
| T4～T5 | `1` | `OKAY` | 不再有 B 的有效数据传输 | Subordinate 对 `IDLE` 给出零等待 `OKAY` |

### 4.1 图中的三个响应阶段

把 T1～T4 的组合单独列出，可以直接看到完整错误序列：

```text
T1-T2  (HRESP,HREADY) = (0,0)  // 决定报错前的普通等待
T2-T3  (HRESP,HREADY) = (1,0)  // ERROR 第一个周期
T3-T4  (HRESP,HREADY) = (1,1)  // ERROR 第二个周期，错误完成
```

Figure 5-1 使用的是 Manager 可见的系统级 `HREADY`。若在目标 Subordinate 本地接口检查同一响应，应观察它输出的 `HREADYOUT`，并由 Interconnect 将该局部返回选择到系统端。

### 4.2 为什么地址 B 与 `Data(A)` 会同时出现

A 是写传输，A 的数据阶段从 T1 开始。Figure 5-1 在 T1～T3 标出的写数据仍是 `Data(A)`，而地址线上已经显示下一笔访问意图 B。这是地址阶段和数据阶段的正常流水线重叠，不表示 `Data(A)` 已经变成 B 的写数据。

> **理解提示：** 同一周期看到地址 B 与写数据 A 是正常的流水线重叠。判断信号归属时必须区分地址阶段和数据阶段，不能把同周期的所有信号都归给 B。

</details>

<a id="5-error-之后的-burst-与读数据"></a>
<details>
<summary><strong>5. <code>ERROR</code> 之后的 Burst 与读数据</strong></summary>

### 5.1 Manager 可以取消剩余 Burst，也可以继续

Manager 收到一笔传输的 `ERROR` 后，可以取消当前 Burst 的剩余传输，但规范不强制取消；继续发送剩余传输同样合法。

必须把三个对象分开：

| 对象 | 能否因当前 `ERROR` 取消 | 原因 |
| --- | --- | --- |
| 正在返回 `ERROR` 的当前传输 | 不能 | 它已经开始，必须以两周期 `ERROR` 完成 |
| 流水线中的下一笔访问意图 | 可以 | Manager 可在错误响应窗口把 `HTRANS` 改为 `IDLE` |
| Burst 中更后面的剩余传输 | 可以取消，也可以继续 | Chapter 5 明确允许 Manager 选择 |

因此，不能把“Manager 可以取消剩余 Burst”误解成“Manager 可以撤回已经被接受的当前传输”。

### 5.2 读错误时的 `HRDATA`

Manager 收到读传输的 `ERROR` 后，仍可能读取甚至使用同周期的 `HRDATA`。Subordinate 不能依靠 `ERROR` 响应来阻止 Manager 获取某个数值。

规范建议在读传输返回 `ERROR` 时把 `HRDATA` 驱动为零。这里是<span style="color:#ffb454"><strong>推荐行为</strong></span>，不是 Manager 可以据此把错误数据当作有效读结果的保证。

> **工程推断：** 为避免泄露旧数据、未初始化数据或受保护位置的内容，安全敏感的 Subordinate 通常应在错误读响应中返回确定的无敏感值，并仍以 `HRESP=1` 表明访问失败。具体数据策略需要由组件安全要求定义。

</details>

<a id="6-实现与验证检查点"></a>
<details>
<summary><strong>6. 实现与验证检查点</strong></summary>

### 6.1 Subordinate 实现检查点

1. 对每笔已接受的有效传输，最终必须给出成功或错误完成，不能永久停留在等待状态。
2. 普通等待周期输出 `(HRESP,HREADYOUT)=(0,0)`。
3. 成功完成输出 `(0,1)`。
4. 错误完成严格输出 `(1,0)` 后紧跟 `(1,1)`；额外等待只能放在该序列之前并保持 `HRESP=0`。
5. 为可变延迟操作定义预期的最大等待周期，便于计算系统最坏延迟并设置验证超时。
6. 读错误时为 `HRDATA` 提供确定值；若采用零值，应把它作为接口策略记录下来。

### 6.2 Manager 实现检查点

1. `HREADY=0` 时不能把 `HRESP=0` 当作传输已经成功。
2. 只有在 `HREADY=1` 的完成边沿，才根据 `HRESP` 判断当前传输成功还是失败。
3. 看到第一个 `ERROR` 周期后，可以把下一笔访问的 `HTRANS` 改为 `IDLE`，但必须让当前传输走完第二个 `ERROR` 周期。
4. 对 Burst 明确定义错误策略：取消剩余拍或继续剩余拍，两者都必须符合 Chapter 3 的传输类型规则。

### 6.3 断言与覆盖建议

验证时可以分别覆盖两条核心序列：

```text
成功路径： (0,0)* -> (0,1)
错误路径： (0,0)* -> (1,0) -> (1,1)
```

推荐至少检查：

- `(1,1)` 不能作为孤立的单周期错误出现；
- `(1,0)` 的下一周期必须保持 `HRESP=1` 并令完成状态为 1；
- 错误前的任意附加等待周期都保持 `HRESP=0`；
- Figure 5-1 类型的错误取消场景中，下一地址阶段切换为 `IDLE`，而当前错误传输仍正常完成；
- 零等待成功、多个等待后成功、无前置等待的错误、有前置等待的错误均被覆盖；
- 等待周期达到组件定义的最大值时，传输能够完成或按组件约定进入错误处理。

> **工程推断：** 若验证环境位于 Subordinate 引脚侧，断言使用 `HREADYOUT`；若位于 Manager 或系统总线侧，断言使用选择后的 `HREADY`。两种视角可以各自验证，但同一条状态序列必须使用同一层次的完成信号。

</details>

<a id="7-易错点口诀与自测"></a>
<details>
<summary><strong>7. 易错点、口诀与自测</strong></summary>

### 7.1 易错点

1. **把 `HRESP=0` 直接当作成功。** 普通等待期间同样必须输出 `HRESP=0`，还要检查完成状态。
2. **把 `HREADY=1` 直接当作成功。** `HREADY=1, HRESP=1` 表示错误完成。
3. **用单周期 `(1,1)` 报错。** AHB 的 `ERROR` 必须先经过 `(1,0)`，再以 `(1,1)` 结束。
4. **在错误判定期间一直保持 `(1,0)`。** 额外等待应使用 `(0,0)` 并放在两周期错误序列之前。
5. **认为两周期 `ERROR` 取消了当前传输。** 当前传输仍以错误完成；取消窗口针对下一笔访问意图。
6. **认为 Burst 出错后必须停止。** Manager 可以取消剩余传输，也可以继续。
7. **认为读错误时 `HRDATA=0` 是强制要求。** 规范推荐驱动为零，但核心协议结果仍由 `HRESP=1` 表示。
8. **混淆 `HREADYOUT` 与 `HREADY`。** 前者是 Subordinate 局部输出，后者是系统级完成状态。
9. **把 `RETRY/SPLIT` 加入本章响应表。** 它们不属于 IHI 0033C Issue C 的 1 bit `HRESP`。

### 7.2 记忆口诀

> <span style="color:#63d297"><strong>结果看 RESP，完成看 READY；普通等用零零，报错必须一零接一一。</strong></span>

<a id="chapter5-self-test"></a>

### 7.3 自测题

<details>
<summary>1. 只看到 <code>HRESP=0</code>，能否断定传输已经成功完成？</summary>

> 不能。`HRESP=0` 表示 `OKAY`，既可能是仍在等待的 `(HRESP,HREADYOUT)=(0,0)`，也可能是成功完成的 `(0,1)`。必须同时检查 `HREADYOUT`；从 Manager 侧则检查系统级 `HREADY`。

</details>

<details>
<summary>2. <code>HRESP</code> 与 <code>HREADYOUT</code> 的四种组合分别表示什么？</summary>

> `(0,0)` 表示普通等待，`(0,1)` 表示成功完成，`(1,0)` 表示 `ERROR` 第一个周期，`(1,1)` 表示 `ERROR` 第二个周期并错误完成。

</details>

<details>
<summary>3. 为什么 <code>ERROR</code> 不能在一个周期内直接完成？</summary>

> 因为 AHB 使用流水线。当前传输开始报错时，下一笔地址通常已经广播。第一个 `ERROR` 周期用 `HREADY=0` 提供取消下一笔访问的时间，第二个周期再以 `HREADY=1, HRESP=1` 结束当前错误传输。

</details>

<details>
<summary>4. Subordinate 需要三个额外周期才能确定错误时，应如何输出响应？</summary>

> 先输出三个普通等待周期 `(HRESP,HREADYOUT)=(0,0)`，然后输出固定的 `(1,0) -> (1,1)` 两周期 `ERROR`。不能提前把三个等待周期都写成 `(1,0)`。

</details>

<details>
<summary>5. 两周期 <code>ERROR</code> 是否允许 Manager 取消正在报错的当前传输？</summary>

> 不允许。当前传输已经开始，必须以 `ERROR` 完成。Manager 可以利用两周期窗口取消流水线中的下一笔访问意图，并可选择是否取消 Burst 的更后续传输。

</details>

<details>
<summary>6. Burst 中一拍返回 <code>ERROR</code> 后，Manager 是否必须取消剩余拍？</summary>

> 不必须。规范允许 Manager 取消剩余传输，也允许继续剩余传输；具体策略由 Manager 和系统设计决定。

</details>

<details>
<summary>7. 读传输返回 <code>ERROR</code> 时，Subordinate 是否必须把 <code>HRDATA</code> 驱动为零？</summary>

> 规范推荐驱动为零，但不是强制的有效数据要求。Manager 仍可能读取该值，Subordinate 不能依赖 `ERROR` 阻止数据被观察；安全敏感设计应返回确定的无敏感值。

</details>

<details>
<summary>8. 为什么通常要为 Subordinate 预先规定最大等待周期？</summary>

> 这样可以计算最坏访问延迟，并防止长时间等待无限制地拖慢整个 AHB 接口。验证环境也可以据此设置合理的超时检查。

</details>

<details>
<summary>9. IHI 0033C Issue C 的 <code>HRESP</code> 是否支持 <code>RETRY</code> 和 <code>SPLIT</code>？</summary>

> 不支持。本规范中的 `HRESP` 为 1 bit，只编码 `OKAY` 和 `ERROR`。`RETRY`、`SPLIT` 属于旧版完整 AHB 的响应概念，不能加入本章状态表。

</details>

</details>

<a id="8-问题记录与解答"></a>

## 8. 问题记录与解答

当前暂无来自本章实际阅读过程的问题记录。

后续如在阅读响应组合、Figure 5-1 或错误后的 Burst 行为时产生具体问题，应在实际引发问题的位置添加入口，并在本节为每个问题建立独立锚点和折叠答案。

<a id="9-本章边界与资料来源"></a>
<details>
<summary><strong>9. 本章边界与资料来源</strong></summary>

本章只解释 Subordinate 的通用传输响应，不展开以下主题：

- `HRESP`、`HREADYOUT`、`HREADY` 的接口方向和 Interconnect 选择关系：参见[第二章信号描述精读](ARM_AMBA_AHB_Protocol_Specification_IHI0033C-第二章信号描述.md)和[第四章总线互连精读](ARM_AMBA_AHB_Protocol_Specification_IHI0033C-第四章总线互连.md)；
- 地址阶段、数据阶段、等待期间信号保持和 Burst 传输类型规则：参见[第三章 Transfers 基础内容精读](ARM_AMBA_AHB_Protocol_Specification_IHI0033C-第三章传输.md)；
- `HWDATA`、`HRDATA`、不同数据总线宽度和端序：继续阅读[第六章 Data Buses 数据总线精读](ARM_AMBA_AHB_Protocol_Specification_IHI0033C-第六章数据总线.md)；
- 复位及不同传输状态下的完整信号有效性要求：继续阅读 Chapter 8 Signal Validity；
- AHB5 Exclusive Transfers 的附加响应 `HEXOKAY`：继续阅读 Chapter 10 Exclusive Transfers；
- 具体错误来源、超时数值、Burst 错误恢复策略和安全数据清零策略：由组件或系统规格定义。

资料来源：

- [AMBA AHB Protocol Specification, Arm IHI 0033C](ARM_AMBA_AHB_Protocol_Specification_IHI0033C.pdf)，Issue C，ID090921，2021 年 9 月 15 日；本文精读 Chapter 5，PDF 第 59-62 页（文档页码 5-59～5-62）。
- 本文对流水线、等待期间信号变化和 Interconnect 返回选择的解释同时参考同一规范 Chapter 3 与 Chapter 4；相关段落已标明超出 Chapter 5 的阅读边界。
- `HEXOKAY` 仅按 Chapter 5 的交叉引用指出其存在，具体语义仍以同一规范 Chapter 10 为准。

</details>

---

本文是对 Arm IHI 0033C 第五章 Subordinate Response Signaling 的中文学习整理，不替代官方规范。实现或验证 AHB 兼容设计时，应以原始 PDF 的规范性描述为准。
