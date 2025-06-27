# 📘 LKS32MC07x MCU MCPWM 学习笔记

> 本笔记记录本人在学习凌鸥 LKS32MC07x 系列 MCU MCPWM 模块过程中的要点、配置方法、实验记录及常见问题总结。

---
## 1. 模块简介
- **概述**：MCPWM
    - 包含两个16位递增计数器，提供了两个时基，计数器的时钟分频有1/2/4/8四种选，产生96MHz/48MHz/24MHz/12MHz,通道0/1/2固定使用时基0，通道3/4/5固定使用时基1；
    - 有6个PWM生成子模块
        - 产生6对互补信号或12路独立不交叠的PWM信号；
        - 支持边沿对齐PWM
        - 中心对齐PWM
        - 移相PWM
        - 同时可以产生 4 路与 MCPWM 同时基的定时信息，用于触发 ADC 模块同步采样，进行与MCPWM 的联动。
        - 包含一组急停保护模块，用于不依赖 CPU 软件的处理快速关断 MCPWM 模块输出。MCPWM模块可输入 4 路急停信号，其中 2 路来自芯片 IO，2 路来自片内比较器的输出。
---
## 2. Base Counter模块
- 由两个递增计数器组成，计数门限值为MCPWM0_TH0/MCPWM0_TH1。计数器从 t0 时刻开始从-TH 递增计数，在 t1 时刻过 0 点，在 t2 时刻计数到 TH 完成一次计数循环，回到-TH，重新开始计数。计数周期为(TH×2+1)×计数时钟周期。
---
## 3.MCPWM波形输出-中心对齐模式
- 有4 个 MCPWM IO Driver 采用独立的控制门限，独立死区宽度（每一对互补 IO 的死区需要独立配置，即 4 个死区配置寄存器），共享数据更新事件。
---
## 4.急停信号的处理
- **信号来源** 
    - MCPWM_InitStructure.GPIO_BKIN_Filter = 12; // IO口急停信号滤波窗口；
    - MCPWM_InitStructure.CMP_BKIN_Filter  = 12; // 比较器急停信号滤波窗口
- 只有当 Fail 输入信号（IO 或 CMP）在 MCPWM 模块中连续保持异常状态达 12+1 = 13 个滤波周期（固定乘16） 后，才认为是一次有效的“急停事件”并触发保护机制。

---
## 5.滤波时间计算
    - 滤波时间的计算公式如下：
$$
T_{\text{filter}} = T_{\text{MCLK}} \times (\text{CLK\_DIV}) \times (\text{FLT\_CLKDIV} + 1) \times 16
$$
 其中：
- $T_{\text{MCLK}}$：系统主时钟周期（如 MCLK = 96MHz，则 $T_{\text{MCLK}}$ = 10.4ns）
- $\text{CLK\_DIV}$：一级分频因子（取值为 1/2/4/8）
- $\text{FLT\_CLKDIV}$：二级滤波分频因子（范围为 0~15）
- `×16`：表示滤波器需检测连续 16 个时钟周期为异常状态才触发 Fail

#### 示例：

若设置如下参数：

- MCLK = 96MHz，即 $T_{\text{MCLK}}$ = 10.4ns
- CLK_DIV = 1
- FLT_CLKDIV = 11（即 `MCPWM_InitStructure.CMP_BKIN_Filter = 11`）

代入公式计算：

$$
T_{\text{filter}} = 10.4\text{ns} \times 1 \times (11 + 1) \times 16 = 10.4\text{ns} \times 12 \times 16 = 1996.8\text{ns} ≈ 2.0\mu\text{s}
$$

因此，该设置下，只有当故障信号持续**大于 2μs**时，MCPWM 才会认定为有效急停并触发保护逻辑。


---