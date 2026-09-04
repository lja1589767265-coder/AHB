# AMBA AHB Protocol Specification IHI 0033C：第八章 Signal validity 信号有效性精读

> 原始资料：[AMBA AHB Protocol Specification IHI 0033C](ARM_AMBA_AHB_Protocol_Specification_IHI0033C.pdf)  
> 精读范围：Chapter 8 Signal validity，PDF 第 73～76 页（文档页码 8-73～8-76）  
> 文档版本：Issue C，ID090921  
> 发布日期：2021 年 9 月 15 日  
> 前置阅读：[第七章 Clock and Reset 时钟与复位精读](ARM_AMBA_AHB_Protocol_Specification_IHI0033C-第七章时钟与复位.md)

本文面向第一次系统学习 AHB 的读者，统一使用 Issue C 的术语：`Manager`（原 `Master`）、`Subordinate`（原 `Slave`）和 `Interconnect`（互连）。第八章篇幅很短，核心任务是回答三个问题：每类信号在什么条件下必须有效、无需有效时发送方可以怎样驱动，以及“有效”“稳定”和“被接收”有什么区别。

> **颜色说明：** 绿色文字表示可直接用于设计与验证的结论；橙色文字表示容易误读的限制。
>
> **信息类型：** 正文用于忠实转述 Chapter 8 的要求；“理解提示”用于连接前文章节；“工程推断”用于说明实现或验证时如何应用这些规则。

<details>
<summary><strong>1. “信号必须有效”究竟是什么意思</strong></summary>

当规范要求一个信号有效时，接收方可以在该条件下依赖它取得确定且符合协议定义的值。对于多位编码信号，这还意味着不能用 `X`、`Z` 或保留编码代替应提供的协议信息。

反过来，当信号**不要求有效**时：

- 发送方可以让它取任意值；
- 规范推荐驱动为 `0` 或 `X`，但这只是推荐，不是强制要求；
- 接收方不能采信该值，也不能根据它作出协议动作；
- “不要求有效”不等于信号必须为 `X`，更不等于对应物理端口必然不存在。

在 RTL 仿真中，驱动 `X` 有助于暴露接收方错误使用无效信号的问题；在综合实现中，设计者还要结合功耗、时序、门级仿真和安全要求选择实际驱动方式。因此，不能把规范中的“推荐 `0` 或 `X`”机械改写成“所有无效周期必须赋 `X`”。

### 无效 byte lane 为什么推荐清零

一笔窄传输只使用数据总线中的部分 byte lane。Chapter 8 建议把未使用的 byte lane 驱动为 `0`，避免接收方意外采样这些 lane 时观察到其他传输遗留的数据，从而在事务之间泄露数据。

这条规则的强制程度要分清：

- **必须：** 活动 byte lane 在规定的有效窗口内提供正确数据；
- **推荐：** 非活动 byte lane 驱动为 `0`；
- **接收方责任：** 不能依赖非活动 byte lane 的值。

> <span style="color:#63d297"><strong>结论：</strong>“不要求有效”表示接收方不得使用该值，而不是发送方必须输出某个固定值；对于无效 byte lane，清零是防止跨事务数据泄露的推荐做法。</span>

### 有效、稳定、被接收是三件不同的事

| 概念 | 回答的问题 | 典型判断条件 |
| --- | --- | --- |
| 有效（valid） | 当前值能否被当作符合协议定义的信息 | 由本章列出的信号分组和条件决定 |
| 稳定（stable） | 延长传输时，相邻采样边沿上的值能否改变 | 由等待状态规则及 Chapter 7 的时钟语义决定 |
| 被接收或完成（accepted/completed） | 当前传输是否在这个上升沿推进或结束 | 通常检查 `HREADY=1`，成功还要同时检查 `HRESP=0` |

例如，写传输进入数据阶段后，`HWDATA` 必须有效；如果 `HREADY=0`，它还必须在等待期间保持稳定，但这笔传输尚未完成。可见“有效”既不自动表示“稳定”，也不自动表示“已经被接收”。

> <span style="color:#ffb454"><strong>易错点：</strong>不要只看到一个信号是确定的 `0/1` 就认为它在当前周期具有协议意义；必须先检查该信号所属的阶段和有效条件。</span>

</details>

<details>
<summary><strong>2. Chapter 8 的五类信号有效条件</strong></summary>

规范把信号分成五组。下表保持原文的条件边界，并把各组对应的协议阶段集中到一起：

| 类别 | 必须有效的条件 | 信号 | 主要含义 |
| --- | --- | --- | --- |
| 1 | 始终有效 | `HTRANS`、`HADDR`、`HSEL`、`HMASTLOCK`、`HREADY`、`HREADYOUT`、`HRESP` | 基本传输状态、地址、选择、锁定和返回控制必须始终是确定值 |
| 2 | `HTRANS!=IDLE` | `HBURST`、`HPROT`、`HSIZE`、`HNONSEC`、`HEXCL`、`HMASTER`、`HWRITE`、`HAUSER` | 当前非空闲地址阶段的属性 |
| 3 | 写事务的数据阶段 | `HWDATA`、`HWSTRB`、`HWUSER` | 写数据及其逐字节和用户属性 |
| 4 | 写事务的数据阶段，并且 `HREADY=1`、`HRESP=0` | `HEXOKAY`、`HBUSER` | 成功完成写事务时的额外返回信息 |
| 5 | 读事务的数据阶段，并且 `HREADY=1`、`HRESP=0` | `HRDATA`、`HEXOKAY`、`HRUSER`、`HBUSER` | 成功完成读事务时的数据及额外返回信息 |

表中的 `HSEL` 是规范在 Chapter 8 使用的统称。在具体多 Subordinate 系统中，通常表现为每个 Subordinate 独立的 `HSELx`。

> <span style="color:#63d297"><strong>结论：</strong>先按“始终有效、非 <code>IDLE</code> 地址阶段、写数据阶段、成功写完成、成功读完成”五层条件分类，就能确定每个信号何时可被接收方依赖。</span>

### 有效条件不决定信号是否必须存在

Chapter 8 说明的是**已有信号何时必须有效**，不是接口端口的强制存在列表。例如：

- `HNONSEC` 由 `Secure_Transfers` 属性控制；
- `HEXCL`、`HMASTER` 和 `HEXOKAY` 由 `Exclusive_Transfers` 属性控制；
- `HWSTRB` 由 `Write_Strobes` 属性控制；
- `HAUSER`、`HWUSER`、`HRUSER` 和 `HBUSER` 是由相应 User 信号宽度属性控制的可选条件信号；
- `HBURST`、`HMASTLOCK` 和某些宽度配置下的 `HPROT` 也可能不出现在特定接口上。

如果接口没有实现某个可选或条件信号，自然不存在对该端口进行有效性检查的问题；如果实现了，就必须遵守本章给出的有效窗口。未被驱动的可选输入应采用 Appendix A 规定的默认值。

> <span style="color:#ffb454"><strong>易错点：</strong>不能因为某个可选信号出现在 Chapter 8 的列表中，就断言所有 AHB 接口都必须具有该端口。</span>

</details>

<details>
<summary><strong>3. 如何逐组理解这些条件</strong></summary>

### 3.1 始终有效：确定值不等于有效传输

`HTRANS`、`HADDR`、`HSEL`、`HMASTLOCK`、`HREADY`、`HREADYOUT` 和 `HRESP` 在每个周期都必须有效。这里最容易混淆的是 `HADDR` 和 `HSEL`：

- `HADDR` 始终是确定的地址值，不代表每个周期都在访问该地址；
- `HSELx` 始终必须是确定的选择值，但 Subordinate 是否接受一笔传输还要结合 `HTRANS` 和 `HREADY`；
- `HMASTLOCK` 始终有效，不代表它始终被断言；
- `HREADY`、`HREADYOUT` 和 `HRESP` 始终有效，分别承担系统完成、局部完成和响应类型的控制作用。

因此，`HTRANS=IDLE` 时 `HADDR` 仍然必须有效，但地址不构成一笔需要执行的传输。复位期间同样要遵守第七章的要求：Manager 令 `HTRANS=IDLE`，Subordinate 令 `HREADYOUT=1`，相关地址和控制输出保持确定的合法值。

> <span style="color:#63d297"><strong>结论：</strong>“始终有效”要求信号始终可解释；是否存在真实传输仍由 <code>HTRANS</code> 的传输类型决定。</span>

### 3.2 `HTRANS!=IDLE` 包含 `BUSY`

第二组条件的原意是“`HTRANS` 不是 `IDLE`”，因此它覆盖 `BUSY`、`NONSEQ` 和 `SEQ`，不能简化为 `HTRANS[1]=1`。

| `HTRANS` | 编码 | 第二组地址属性必须有效 | 是否表示一笔真实数据传输 |
| --- | --- | --- | --- |
| `IDLE` | `2'b00` | 否 | 否 |
| `BUSY` | `2'b01` | 是 | 否 |
| `NONSEQ` | `2'b10` | 是 | 是 |
| `SEQ` | `2'b11` | 是 | 是 |

`HTRANS[1]` 可以区分“真实传输”与 `IDLE/BUSY`，却不能表达本章的“非 `IDLE`”有效性条件，因为 `BUSY` 的高位也是 `0`。设计或验证中应分别写出两个判断：

```systemverilog
addr_attr_valid_required = (HTRANS != 2'b00); // BUSY、NONSEQ、SEQ
real_transfer            =  HTRANS[1];         // NONSEQ、SEQ
```

`BUSY` 不会创建新的数据阶段，但它处于 Burst 上下文中，所以本章仍要求 `HBURST`、`HPROT`、`HSIZE`、`HWRITE` 等第二组信号有效。具体 Burst 稳定性和 `BUSY` 地址规则仍以 Chapter 3 为准。

> <span style="color:#63d297"><strong>结论：</strong>判断第二组信号是否必须有效要使用 <code>HTRANS!=IDLE</code>；判断是否发起真实传输才可以使用 <code>HTRANS[1]</code>。</span>

### 3.3 写数据在等待期间仍必须有效

`HWDATA`、`HWSTRB` 和 `HWUSER` 的有效窗口覆盖一笔写事务的整个数据阶段。这个条件没有附加 `HREADY=1` 或 `HRESP=0`，所以：

- 无等待的写事务中，它们在唯一的数据周期有效；
- `HREADY=0` 延长数据阶段时，它们在等待周期仍必须有效，并按各自章节的稳定性规则处理；其中 User 信号还要保留 Chapter 11 对 `ERROR` 响应的例外；
- 写事务最终返回 `ERROR` 时，不能据此倒推此前的写数据信号可以无效。

对于窄传输，`HWDATA` 只要求活动 byte lane 含有正确写数据，非活动 lane 推荐清零。启用 `HWSTRB` 时，它还明确指出哪些 byte lane 参与写入。

> <span style="color:#63d297"><strong>结论：</strong>写数据组的有效条件是“处于写数据阶段”，不是“写事务正在成功完成”。</span>

### 3.4 返回数据与附加响应只在成功完成时有效

对最后两组信号，必须同时满足三个条件：

1. 当前数据阶段的方向正确，是读或写；
2. `HREADY=1`，传输在本周期完成；
3. `HRESP=0`，传输以 `OKAY` 响应成功完成。

各场景可归纳如下：

| 当前数据阶段 | `HREADY` | `HRESP` | 必须有效的方向相关信号 |
| --- | ---: | ---: | --- |
| 写等待 | `0` | `0` | `HWDATA`、`HWSTRB`、`HWUSER` |
| 写成功完成 | `1` | `0` | 写数据组，以及 `HEXOKAY`、`HBUSER` |
| 写错误响应 | 依错误响应周期而定 | `1` | 写数据组；不能要求 `HEXOKAY`、`HBUSER` 有效 |
| 读等待 | `0` | `0` | 不能要求 `HRDATA`、`HEXOKAY`、`HRUSER`、`HBUSER` 有效 |
| 读成功完成 | `1` | `0` | `HRDATA`、`HEXOKAY`、`HRUSER`、`HBUSER` |
| 读错误完成 | `1` | `1` | 不能要求读数据和附加返回组有效 |

`HREADY=1` 只说明传输可以结束，不能单独证明成功；必须再检查 `HRESP=0`。这也是为什么错误读响应中的 `HRDATA` 不可使用。

`HEXOKAY` 有效也不等于它一定为 `1`。它是 Exclusive Transfer 的附加响应：在有效窗口内必须给出确定值，只有符合 Chapter 10 条件的 Exclusive Transfer 成功时才允许断言。`HBUSER` 则是读写共用的 User response。

> <span style="color:#63d297"><strong>结论：</strong>读数据和附加返回信息只在 <code>HREADY=1</code>、<code>HRESP=0</code> 的成功完成周期可用；仅检查 <code>HREADY</code> 会把错误响应误判为有效结果。</span>

</details>

<details>
<summary><strong>4. 流水线中应分别判断地址阶段和数据阶段</strong></summary>

AHB 的地址阶段与数据阶段可以重叠。同一个时钟周期中，地址和控制信号可能属于传输 B，而数据和返回信号属于传输 A。因此，本章的不同条件不能全部使用同周期的 `HWRITE`、`HTRANS` 或 `HSELx` 解释。

判断一个周期内各信号是否必须有效时，可以按以下顺序进行：

1. **先看接口是否实现该信号。** 未实现的可选信号跳过，已实现的信号继续判断。
2. **判断当前地址阶段。** `HTRANS`、`HADDR`、`HSELx` 和 `HMASTLOCK` 始终有效；当 `HTRANS!=IDLE` 时，第二组地址属性也必须有效。
3. **确定当前数据阶段来自哪一笔传输。** 使用前一笔已接受地址阶段保存下来的方向和属性，不能直接借用同周期下一笔地址阶段的 `HWRITE`。
4. **如果当前是写数据阶段，**要求 `HWDATA`、`HWSTRB` 和 `HWUSER` 有效；等待期间继续保持。
5. **如果当前传输成功完成，**再根据保存的读写方向检查读返回组或写返回组。

下面的伪代码展示这种分层方式，其中 `data_phase_valid` 和 `data_phase_write` 必须由已经接受的地址阶段产生，并在 `HREADY=0` 时保持：

```systemverilog
addr_attr_required     = (HTRANS != 2'b00);
write_data_required    = data_phase_valid &&  data_phase_write;
write_return_required  = write_data_required && HREADY && !HRESP;
read_return_required   = data_phase_valid && !data_phase_write
                       && HREADY && !HRESP;
```

> **工程推断：** 验证环境最好为每笔被接受的 `NONSEQ/SEQ` 地址阶段建立事务状态，再用该状态检查后续数据阶段。只用当前总线上的 `HWRITE` 判断 `HRDATA` 或 `HWDATA` 的有效性，会在连续读写切换和等待状态下把两笔传输混在一起。

> <span style="color:#63d297"><strong>结论：</strong>地址属性的有效性由当前地址阶段的 <code>HTRANS</code> 决定；数据与返回信号的有效性由已经进入数据阶段的上一笔传输决定。</span>

</details>

<details>
<summary><strong>5. 设计与验证检查清单</strong></summary>

### RTL 设计侧

- 始终为 `HTRANS`、`HADDR`、`HSELx`、`HMASTLOCK`、`HREADY`、`HREADYOUT` 和 `HRESP` 提供确定的合法值。
- `HTRANS!=IDLE` 时，所有已实现的第二组地址属性都为有效值；不要漏掉 `BUSY`。
- 用已接受地址阶段保存的读写方向控制数据阶段，不用同周期下一笔传输的 `HWRITE` 代替。
- 写数据阶段始终驱动有效的 `HWDATA`，以及接口已经启用的 `HWSTRB`、`HWUSER`；等待期间满足稳定性要求。
- 只有成功读完成时才消费 `HRDATA`、`HRUSER` 和读侧 `HBUSER`。
- 只有成功完成时才消费 `HEXOKAY` 和 `HBUSER`，同时遵守 Exclusive 与 User signaling 章节的额外规则。
- 非活动 byte lane 优先清零，避免无效数据残留和不必要的切换。
- 对不存在的可选信号使用接口属性和默认值处理，不伪造无意义端口。

### 验证侧

- 对第一组信号建立逐周期的 unknown 和非法编码检查，包括 `IDLE` 与复位窗口。
- 第二组断言的门控条件写成 `HTRANS!=IDLE`，并覆盖 `BUSY` 场景。
- 使用数据阶段事务跟踪器检查 `HWDATA`、`HRDATA` 和方向相关 User 信号，避免把地址阶段与数据阶段错位。
- 分别覆盖读等待、读成功、读错误、写等待、写成功和写错误。
- 在 `HREADY=0` 的延长传输中，同时检查“仍然有效”和“按协议保持稳定”。
- 如果项目把“无效 byte lane 清零”提升为内部安全规范，应单独建立检查；不要把 Chapter 8 的推荐误报为 AHB 协议强制错误。
- 对可选信号按接口属性生成断言，避免引用不存在的端口或错误使用默认值。

</details>

<details>
<summary><strong>6. 易错点与记忆方法</strong></summary>

### 六个易错点

1. **把有效等同于传输成立。** `HADDR` 始终有效，但 `HTRANS=IDLE` 时没有真实传输。
2. **把 `HTRANS!=IDLE` 写成 `HTRANS[1]=1`。** 这样会漏掉 `BUSY`。
3. **只在写完成周期检查 `HWDATA`。** 写数据组在整个写数据阶段都必须有效。
4. **在读等待周期使用 `HRDATA`。** 读返回组只在 `HREADY=1`、`HRESP=0` 时有效。
5. **看到 `HREADY=1` 就认定返回数据有效。** 错误响应也会完成，必须同时检查 `HRESP=0`。
6. **把可选信号的有效性规则当成端口存在规则。** 信号是否出现由接口属性决定，出现后才应用本章条件。

### 一句话记忆

> 常驻信号始终真，地址属性非空闲；写数覆盖数据段，读回只取成功点。

其中“非空闲”包含 `BUSY`，“成功点”指 `HREADY=1` 且 `HRESP=0`。

</details>

<details>
<summary><strong>7. 自测题</strong></summary>

<details>
<summary>1. 信号不要求有效时，发送方是否必须驱动 <code>X</code>？</summary>

> 不必须。规范允许它取任意值，只是推荐驱动为 `0` 或 `X`；接收方不得依赖该值。

</details>

<details>
<summary>2. 为什么无效 byte lane 推荐驱动为 <code>0</code>？</summary>

> 为了避免无效 lane 被意外采样时暴露其他事务遗留的数据，减少跨事务数据泄露风险。它是推荐做法，不改变接收方只能使用活动 lane 的责任。

</details>

<details>
<summary>3. <code>HADDR</code> 始终有效，是否说明每个周期都有地址传输？</summary>

> 不是。`HADDR` 始终为确定值，但是否存在真实传输要看 `HTRANS`；`IDLE` 和 `BUSY` 都不产生真实数据传输。

</details>

<details>
<summary>4. 哪些 <code>HTRANS</code> 取值要求第二组地址属性有效？</summary>

> `BUSY`、`NONSEQ` 和 `SEQ`，因为三者都满足 `HTRANS!=IDLE`。只有 `IDLE` 不要求第二组信号有效。

</details>

<details>
<summary>5. 为什么不能用 <code>HTRANS[1]</code> 作为第二组信号的有效门控？</summary>

> 因为 `BUSY=2'b01`，其 `HTRANS[1]=0`，但 Chapter 8 仍要求 `BUSY` 周期的第二组地址属性有效。`HTRANS[1]` 适合判断 `NONSEQ/SEQ` 真实传输，不等价于“非 `IDLE`”。

</details>

<details>
<summary>6. 写等待周期中的 <code>HWDATA</code> 是否必须有效？</summary>

> 必须。写数据组在整个写数据阶段都必须有效；`HREADY=0` 延长数据阶段时，还要按协议要求保持稳定。

</details>

<details>
<summary>7. 读传输在什么条件下必须提供有效 <code>HRDATA</code>？</summary>

> 当前必须处于读事务的数据阶段，并且 `HREADY=1`、`HRESP=0`，也就是传输成功完成。

</details>

<details>
<summary>8. <code>HREADY=1</code> 时能否直接使用 <code>HRDATA</code>？</summary>

> 不能。还要确认当前数据阶段是读事务并且 `HRESP=0`；`HREADY=1`、`HRESP=1` 表示错误完成，此时 `HRDATA` 不要求有效。

</details>

<details>
<summary>9. <code>HEXOKAY</code> 出现在有效性列表中，是否说明每个接口都必须实现它？</summary>

> 不是。`HEXOKAY` 由 `Exclusive_Transfers` 属性控制。只有接口实现该信号时，才需要在本章规定的成功完成窗口内保证它有效。

</details>

<details>
<summary>10. 为什么不能用同周期的 <code>HWRITE</code> 判断当前 <code>HRDATA</code> 是否有效？</summary>

> 因为同周期的 `HWRITE` 可能属于下一笔传输的地址阶段，而 `HRDATA` 属于上一笔传输的数据阶段。应使用已接受地址阶段保存下来的数据阶段方向进行判断。

</details>

</details>

## 8. 问题记录与解答

暂无。

<details>
<summary><strong>9. 本篇边界与资料来源</strong></summary>

本篇只解释 Chapter 8 的信号有效窗口，不重复展开每个信号的完整功能、等待状态稳定性、错误响应时序、Exclusive Transfers 或 User signaling。相关内容可继续阅读：

- [第二章 Signal Descriptions 信号描述精读](ARM_AMBA_AHB_Protocol_Specification_IHI0033C-第二章信号描述.md)：核心信号的方向、位宽和接口位置；
- [第三章 Transfers 传输精读](ARM_AMBA_AHB_Protocol_Specification_IHI0033C-第三章传输.md)：`HTRANS`、地址阶段、数据阶段和等待状态；
- [第五章 Subordinate Response Signaling 响应信号精读](ARM_AMBA_AHB_Protocol_Specification_IHI0033C-第五章Subordinate响应信号.md)：`HREADY`、`HRESP` 与成功或错误完成；
- [第六章 Data Buses 数据总线精读](ARM_AMBA_AHB_Protocol_Specification_IHI0033C-第六章数据总线.md)：`HWDATA`、`HRDATA` 和 byte lane；
- [第七章 Clock and Reset 时钟与复位精读](ARM_AMBA_AHB_Protocol_Specification_IHI0033C-第七章时钟与复位.md)：采样边沿、稳定性和复位输出；
- Chapter 9 Atomicity：单副本与多副本原子性；
- Chapter 10、Chapter 11 和 Appendix A：Exclusive、User 信号以及端口存在属性和默认值。

本文以 [AMBA AHB Protocol Specification IHI 0033C](ARM_AMBA_AHB_Protocol_Specification_IHI0033C.pdf) 的 Chapter 8（PDF 第 73～76 页，文档页码 8-73～8-76）为主要资料。关于可选信号的存在条件和默认值，同时参照 Appendix A；关于 `HEXOKAY` 和 User 信号的含义，分别参照 Chapter 10 和 Chapter 11。Chapter 8 没有规范配图，因此本文使用条件表格呈现原文规则，没有额外复制空白续页或制作装饰性图片。

本文是学习整理，不替代官方规范。实现或验证 AHB 兼容设计时，应以原始 PDF 为准。

</details>
