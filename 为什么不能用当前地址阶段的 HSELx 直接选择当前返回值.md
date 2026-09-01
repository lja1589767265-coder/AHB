# 为什么不能用当前地址阶段的 HSELx 直接选择当前返回值

## 1. 先看整体结构

AHB 中，Manager 通过 `HADDR` 发出当前地址，Decoder 根据地址产生对应的 `HSELx`：

```text
                 HADDR
Manager ───────────────→ Decoder
                           │
                           ├── HSEL_1 ──→ Subordinate 1
                           ├── HSEL_2 ──→ Subordinate 2
                           └── HSEL_3 ──→ Subordinate 3


Subordinate 1 ── HRDATA_1 / HREADYOUT_1 / HRESP_1 ─┐
Subordinate 2 ── HRDATA_2 / HREADYOUT_2 / HRESP_2 ─┼→ Multiplexer → Manager
Subordinate 3 ── HRDATA_3 / HREADYOUT_3 / HRESP_3 ─┘
                                         ↑
                              数据阶段的 Slave select
```

关键就在于：**Multiplexer 当前应该选择哪个 Subordinate 的返回值？**

---

## 2. AHB 是流水线传输

AHB 的重要特点是：

- 当前周期进行一笔传输的 **Address Phase**
- 同时进行上一笔传输的 **Data Phase**

例如连续访问：

```text
第 1 笔：访问 Subordinate 1
第 2 笔：访问 Subordinate 2
```

时序可以理解成：

```text
时间              T0                T1                T2
                  │                 │                 │
HCLK          ____↑_________________↑_________________↑____

地址阶段       Address A           Address B
HADDR          Slave 1地址          Slave 2地址

HSEL_1         1                    0
HSEL_2         0                    1

数据阶段                           Data A             Data B
                                  ↑                  ↑
                                 Slave1             Slave2
```

重点看 **T1～T2**。

这时地址阶段已经开始访问 Slave2：

```text
HADDR  = Slave2 地址
HSEL_2 = 1
```

但是数据阶段正在完成上一笔 Slave1 的访问：

```text
HRDATA_1
HREADYOUT_1
HRESP_1
```

也就是说：

```text
              当前地址阶段
                  ↓
Manager → HADDR = Slave2
                  ↓
               Decoder
                  ↓
              HSEL_2 = 1

上一笔数据阶段
                  ↓
Subordinate 1 → HRDATA_1
                  ↓
             Multiplexer
                  ↓
               Manager
```

---

## 3. 为什么不能直接拿当前 HSELx 控制 Multiplexer

假设直接这样连接：

```text
Decoder
   │
 HSEL_1
 HSEL_2
 HSEL_3
   │
   └────────→ Multiplexer select
```

那么在 T1～T2：

```text
HSEL_1 = 0
HSEL_2 = 1
```

Multiplexer 会认为：

```text
HSEL_2 = 1
→ 选择 HRDATA_2
```

于是：

```text
Multiplexer

X  HRDATA_1    ← 实际应该选择
✓  HRDATA_2    ← 却错误选择了这一项
```

但此时真正正在完成的是上一笔 **Slave1** 的传输。

因此正确返回应该是：

```text
HRDATA_1
HREADYOUT_1
HRESP_1
```

而不是 Slave2 的返回值。

---

## 4. 正确做法：保存地址阶段的选择结果

因此，设计里需要区分两种选择信息：

1. **地址阶段选择**
2. **数据阶段选择**

结构可以理解为：

```text
             当前地址阶段
                  │
HADDR ───────→ Decoder
                  │
             HSEL_1/2/3
                  │
                  ├────────→ Subordinate
                  │
                  │   HCLK 上升沿
                  │       ↓
                  └──→ [保存选择结果]
                           │
                           ↓
                  Data-phase select
                           │
                           ↓
                     Multiplexer
```

例如当前地址阶段访问 Slave1：

```text
HADDR = Slave1 地址
           ↓
Decoder
           ↓
HSEL_1 = 1
           ↓
       HCLK ↑
           ↓
保存：

data_sel = Slave1
```

到了下一拍，虽然地址阶段已经访问 Slave2：

```text
HADDR  = Slave2 地址
HSEL_2 = 1
```

但数据阶段仍然应该使用：

```text
data_sel = Slave1
```

所以 Multiplexer 选择：

```text
HRDATA_1
HREADYOUT_1
HRESP_1
```

再下一拍：

```text
data_sel = Slave2
```

于是 Multiplexer 再切换到：

```text
HRDATA_2
HREADYOUT_2
HRESP_2
```

---

## 5. HSEL 和 Data-phase select 的区别

最简单的理解：

> **HSEL 表示“这一拍地址在访问谁”。**

而：

> **Data-phase select 表示“这一拍的数据是谁返回的”。**

因为 AHB 是流水线结构，所以：

```text
这一拍的地址    ≠    这一拍的数据
     ↓                   ↓
   HSEL              data_sel
```

因此不能直接把当前地址阶段的 `HSELx` 当作当前数据阶段 Multiplexer 的选择信号。

---

## 6. HREADY 为什么也很关键

如果没有 Wait State，可以简单理解成：

```text
data_sel = HSEL 延迟一拍
```

但是严格来说，还要考虑 `HREADY`。

假设 Slave1 需要多个周期才能完成：

```text
                 T1        T2        T3
                  │         │         │

Slave1 数据阶段   ├───────────────────┤

HREADY            0         0         1
```

在 `HREADY = 0` 时，当前数据阶段还没有结束。

因此 Multiplexer 必须继续保持：

```text
data_sel = Slave1
```

不能每个时钟都机械地执行：

```systemverilog
data_sel <= HSEL;
```

否则 Slave1 还没有完成，Multiplexer 就可能切到其他 Slave。

更准确的概念逻辑是：

```systemverilog
if (HREADY) begin
    // 当前数据阶段完成
    // 接收新的地址阶段选择结果
    data_sel <= current_hsel;
end
else begin
    // 当前数据阶段没有完成
    // 保持原来的 Slave
    data_sel <= data_sel;
end
```

结构上可以理解为：

```text
                  HREADY = 1
                     │
                     ▼
HADDR → Decoder → HSEL_x → 寄存/保存 → data_sel → MUX
                              ▲
                              │
                       HREADY = 0 时保持
```

---

## 7. 对原搜索结果中一句话的修正

原文有一句：

> HSELx 的值由上一拍的 HADDR 译码产生。

这个说法容易误导。

更准确的是：

> **HSELx 是当前地址阶段 `HADDR` 的译码结果。**

而：

> **当前数据阶段 Multiplexer 使用的选择信息，来自前一个已经被接受的地址阶段。**

因此应该理解成：

```text
当前 HADDR
    ↓
当前 HSEL
    ↓
选择当前地址阶段的 Slave

前一笔地址阶段的 HSEL
    ↓
保存后的 data_sel
    ↓
选择当前数据阶段返回的 Slave
```

---

## 8. 回到 AHB 框图应该怎么画

之前图中的：

```text
Decoder
   │
   └────────→ Multiplexer select
```

如果直接理解成当前的 `HSEL_1/HSEL_2/HSEL_3` 控制 Multiplexer，就不够准确。

更合理的画法应该是：

```text
                    Address Phase
                         │
HADDR ───────────────→ Decoder
                         │
                    HSEL_1/2/3
                         │
             ┌───────────┴───────────┐
             │                       │
             ▼                       ▼
      Subordinate 选择         Data-phase select
                                     │
                              [保存/寄存选择结果]
                                     │
                                     ▼
                               Multiplexer
                                     │
                   ┌─────────────────┼─────────────────┐
                   ▼                 ▼                 ▼
                HRDATA             HREADY            HRESP
                                     │
                                     ▼
                                  Manager
```

---

## 9. 一句话记忆

可以记成：

> **HSEL 管地址阶段，data_sel 管数据阶段。**

或者更形象一点：

> **这一拍 HSEL 选“这一笔地址访问谁”，而 Multiplexer 要选“当前数据阶段是谁在返回”。**

这就是为什么 **不能直接使用当前地址阶段的 HSELx 去选择当前返回值**。
