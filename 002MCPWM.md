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

## 6.MCPWM配置
- **GPIO口配置**
    - 先将结构体参数初始化后，再进行配置；
    - 将GPIO口配置成输出模式，并将其配置成复用3的功能
    ```c
    GPIO_InitStruct.GPIO_Mode = GPIO_Mode_OUT;
    GPIO_InitStruct.GPIO_Pin = GPIO_Pin_4 | GPIO_Pin_5 | GPIO_Pin_6 | GPIO_Pin_7 | GPIO_Pin_8 | GPIO_Pin_9;
    GPIO_Init(GPIO1, &GPIO_InitStruct);
    GPIO_PinAFConfig(GPIO1, GPIO_PinSource_4, AF3_MCPWM);
    GPIO_PinAFConfig(GPIO1, GPIO_PinSource_5, AF3_MCPWM);
    GPIO_PinAFConfig(GPIO1, GPIO_PinSource_6, AF3_MCPWM);
    GPIO_PinAFConfig(GPIO1, GPIO_PinSource_7, AF3_MCPWM);
    GPIO_PinAFConfig(GPIO1, GPIO_PinSource_8, AF3_MCPWM);
    GPIO_PinAFConfig(GPIO1, GPIO_PinSource_9, AF3_MCPWM);
    ```
- **定时器配置**
    - 先将结构体参数初始化后，再进行配置；
    - 配置定时器各个通道的工作模式；
    - 配置定时器捕获方式；
    - 定时器回零时电平的极性控制；
    - 设置定时器的计数值；
    - 设置定时器的各个通道的比较值；
    - 设置捕捉模式或编码器模式下对应通道的数字滤波值；
    - 设置定时器的分频系数；
    - 开启定时器中断使能；有五个参数
        ```c
        /**
        * @brief 中断使能配置定义
        */
        typedef enum
        {
            Timer_IRQEna_None = 0,
            Timer_IRQEna_Zero = BIT0,     /**<使能过零中断*/
            Timer_IRQEna_CH0 = BIT1,      /**<使能CH0中断，含比较、捕获中断*/
            Timer_IRQEna_CH1 = BIT2,      /**<使能CH1中断，含比较、捕获中断*/
            Timer_IRQEna_All = 0x07       /**<使能Timer全部中断*/
        } Timer_IRQEnaDef;
        ```
    - 初始化定时器
    - 定时器模块使能
    - ```c
        void UTimer_init(void)
        {
            TIM_TimerInitTypeDef TIM_InitStruct;  

            TIM_TimerStrutInit(&TIM_InitStruct);
            TIM_InitStruct.Timer_CH0_WorkMode = TIMER_OPMode_CMP;  /* 设置Timer CH0 为比较模式 */
            TIM_InitStruct.Timer_CH0_CapMode = TIMER_CapMode_None;
            TIM_InitStruct.Timer_CH0Output = 0;                    /* 计数器回零时，比较模式输出极性控制 */
            TIM_InitStruct.Timer_CH1_WorkMode = TIMER_OPMode_CMP;  /* 设置Timer CH1 为比较模式 */
            TIM_InitStruct.Timer_CH1_CapMode = TIMER_CapMode_None;
            TIM_InitStruct.Timer_CH1Output = 0;                    /* 计数器回零时，比较模式输出极性控制 */
            TIM_InitStruct.Timer_TH = 48000;                       /* 设置计数器计数模值 */
            TIM_InitStruct.Timer_CMP0 = 24000;                     /* 设置比较模式的CH0比较值 */
            TIM_InitStruct.Timer_CMP1 = 500;
            TIM_InitStruct.Timer_Filter0 = 0;                      /* 设置捕捉模式或编码器模式下对应通道的数字滤波值 */
            TIM_InitStruct.Timer_Filter1 = 0;
            TIM_InitStruct.Timer_ClockDiv = TIM_Clk_Div1;          /* 设置Timer模块数据分频系数 */
            TIM_InitStruct.Timer_IRQEna = Timer_IRQEna_Zero;       /* 开启Timer模块过0中断 */
            TIM_TimerInit(TIMER0, &TIM_InitStruct);
            TIM_TimerCmd(TIMER0, ENABLE);                          /* Timer0 模块使能 */
        }
      ```

- **MCPWM模式配置**
    - 先将结构体参数初始化后，再进行配置；
    - 对MCPWM的时钟可以再进行一次分频；
    - 使能MCPWM模块；
    - 使能MCPWM计数器使能；
    - 配置通道的工作模式，中心或者边沿对齐；
    - 配置滤波器的急停信号的数字滤波；
    - 配置MCPWM的计数周期；
    - 配置MCPWM对于ADC的触发事件中断；
    - 配置三对互补通道的死区时间；
    - 若有预驱配置预驱上下桥臂的输出极性有效值；
    - 配置是否交换互补通道的输出极性；
    - 配置互补通道当出现错误或者MOE模式下的默认输出极性;
    - 配置Debug时，MCU进入Halt, MCPWM信号是否正常输出；
    - 配置MCPWM的T0事件更新使能事件中断使能；
    - 配置FAULT0或者FAULT1中断使能，是否开启急停功能，信号的选择，极性的选择；
    - 更新上述的配置；

    - ```
                void MCPWM_init(void)
        {
            MCPWM_InitTypeDef MCPWM_InitStructure;

            MCPWM_StructInit(&MCPWM_InitStructure);
            
            MCPWM_InitStructure.CLK_DIV = 0;                          /* MCPWM时钟分频设置 */
            MCPWM_InitStructure.MCLK_EN = ENABLE;                     /* 模块时钟开启 */
            MCPWM_InitStructure.MCPWM_Cnt_EN = ENABLE;                /* 主计数器开始计数使能开关 */
            MCPWM_InitStructure.MCPWM_WorkModeCH0 = CENTRAL_PWM_MODE;	/* 通道工作模式设置成中心对齐模式 */
            MCPWM_InitStructure.MCPWM_WorkModeCH1 = CENTRAL_PWM_MODE; /* 通道工作模式设置，中心对齐或边沿对齐 */
            MCPWM_InitStructure.MCPWM_WorkModeCH2 = CENTRAL_PWM_MODE;

                /*  */
            MCPWM_InitStructure.GPIO_BKIN_Filter = 12;                /* 急停事件(来自IO口信号)数字滤波器时间设置 */    /**< GPIO输入滤波时钟设置1-16 */
            MCPWM_InitStructure.CMP_BKIN_Filter = 12;                 /* 急停事件(来自比较器信号)数字滤波器时间设置 */
            

            MCPWM_InitStructure.MCPWM_PERIOD = PWM_PERIOD;            	/* 计数周期设置 */
            MCPWM_InitStructure.TriggerPoint0 = (u16)(10 - PWM_PERIOD); /* MCPWM_TMR0 ADC触发事件T0 设置 */
            MCPWM_InitStructure.TriggerPoint1 = (u16)(PWM_PERIOD - 1);	/* MCPWM_TMR1 ADC触发事件T1 设置 */
            
            
            MCPWM_InitStructure.DeadTimeCH0N = DEADTIME;
            MCPWM_InitStructure.DeadTimeCH0P = DEADTIME;
            MCPWM_InitStructure.DeadTimeCH1N = DEADTIME;
            MCPWM_InitStructure.DeadTimeCH1P = DEADTIME;
            MCPWM_InitStructure.DeadTimeCH2N = DEADTIME;
            MCPWM_InitStructure.DeadTimeCH2P = DEADTIME;              /* 死区时间设置 */

        #if (PRE_DRIVER_POLARITY == P_HIGH__N_LOW)                    /* CHxP 高有效， CHxN低电平有效 */
            MCPWM_InitStructure.CH0N_Polarity_INV = ENABLE;           /* CH0N通道输出极性设置 | 正常输出或取反输出*/
            MCPWM_InitStructure.CH0P_Polarity_INV = DISABLE;          /* CH0P通道输出极性设置 | 正常输出或取反输出 */
            MCPWM_InitStructure.CH1N_Polarity_INV = ENABLE;
            MCPWM_InitStructure.CH1P_Polarity_INV = DISABLE;
            MCPWM_InitStructure.CH2N_Polarity_INV = ENABLE;
            MCPWM_InitStructure.CH2P_Polarity_INV = DISABLE;

            MCPWM_InitStructure.Switch_CH0N_CH0P =  DISABLE;           /* 通道交换选择设置 | CH0P和CH0N是否选择信号交换 */
            MCPWM_InitStructure.Switch_CH1N_CH1P =  DISABLE;           /* 通道交换选择设置 */
            MCPWM_InitStructure.Switch_CH2N_CH2P =  DISABLE;           /* 通道交换选择设置 */

            /* 默认电平设置 默认电平输出不受MCPWM_IO01和MCPWM_IO23的 BIT0、BIT1、BIT8、BIT9、BIT6、BIT14
                                                            通道交换和极性控制的影响，直接控制通道输出 */
            MCPWM_InitStructure.CH0P_default_output = LOW_LEVEL;
            MCPWM_InitStructure.CH0N_default_output = HIGH_LEVEL;
            MCPWM_InitStructure.CH1P_default_output = LOW_LEVEL;      /* CH1P对应引脚在空闲状态输出低电平 */
            MCPWM_InitStructure.CH1N_default_output = HIGH_LEVEL;     /* CH1N对应引脚在空闲状态输出高电平 */
            MCPWM_InitStructure.CH2P_default_output = LOW_LEVEL;
            MCPWM_InitStructure.CH2N_default_output = HIGH_LEVEL;
        #else
        #if (PRE_DRIVER_POLARITY == P_HIGH__N_HIGH)                    /* CHxP 高有效， CHxN高电平有效 */
            MCPWM_InitStructure.CH0N_Polarity_INV = DISABLE;           /* CH0N通道输出极性设置 | 正常输出或取反输出*/
            MCPWM_InitStructure.CH0P_Polarity_INV = DISABLE;          /* CH0P通道输出极性设置 | 正常输出或取反输出 */
            MCPWM_InitStructure.CH1N_Polarity_INV = DISABLE;
            MCPWM_InitStructure.CH1P_Polarity_INV = DISABLE;
            MCPWM_InitStructure.CH2N_Polarity_INV = DISABLE;
            MCPWM_InitStructure.CH2P_Polarity_INV = DISABLE;

            MCPWM_InitStructure.Switch_CH0N_CH0P =  DISABLE;           /* 通道交换选择设置 | CH0P和CH0N是否选择信号交换 */
            MCPWM_InitStructure.Switch_CH1N_CH1P =  DISABLE;           /* 通道交换选择设置 */
            MCPWM_InitStructure.Switch_CH2N_CH2P =  DISABLE;           /* 通道交换选择设置 */

            /* 默认电平设置 默认电平输出不受MCPWM_IO01和MCPWM_IO23的 BIT0、BIT1、BIT8、BIT9、BIT6、BIT14
                                                            通道交换和极性控制的影响，直接控制通道输出 */
            MCPWM_InitStructure.CH0P_default_output = LOW_LEVEL;
            MCPWM_InitStructure.CH0N_default_output = LOW_LEVEL;
            MCPWM_InitStructure.CH1P_default_output = LOW_LEVEL;      /* CH1P对应引脚在空闲状态输出低电平 */
            MCPWM_InitStructure.CH1N_default_output = LOW_LEVEL;     /* CH1N对应引脚在空闲状态输出高电平 */
            MCPWM_InitStructure.CH2P_default_output = LOW_LEVEL;
            MCPWM_InitStructure.CH2N_default_output = LOW_LEVEL;
        #endif
        #endif

            MCPWM_InitStructure.DebugMode_PWM_out = ENABLE;           /* 在接上仿真器debug程序时，暂停MCU运行时，选择各PWM通道正常输出调制信号
                                                                        还是输出默认电平，保护功率器件 ENABLE:正常输出 DISABLE:输出默认电平*/

            MCPWM_InitStructure.MCPWM_T0_UpdateEN = ENABLE;           /* MCPWM T0事件更新使能 */
            MCPWM_InitStructure.MCPWM_T1_UpdateEN = DISABLE;          /* MCPWM T1事件更新 禁止*/

        #if (CURRENT_SAMPLE_TYPE == CURRENT_SAMPLE_1SHUNT)
            MCPWM_InitStructure.T1_Update_INT_EN = ENABLE;           /* T0更新事件 中断使能或关闭 */
        #else
        #if (CURRENT_SAMPLE_TYPE == CURRENT_SAMPLE_2SHUNT)
            MCPWM_InitStructure.T0_Update_INT_EN = DISABLE;           /* T0更新事件 中断使能或关闭 */
        #else
        #if ((CURRENT_SAMPLE_TYPE == CURRENT_SAMPLE_3SHUNT)||(CURRENT_SAMPLE_TYPE == CURRENT_SAMPLE_MOSFET)) 
            MCPWM_InitStructure.T0_Update_INT_EN = DISABLE;           /* T0更新事件 中断使能或关闭 */
        #endif    
        #endif
        #endif

            MCPWM_InitStructure.FAIL0_INT_EN = DISABLE;               /* FAIL0事件 中断使能或关闭 */
            MCPWM_InitStructure.FAIL0_INPUT_EN = DISABLE;             /* FAIL0通道急停功能打开或关闭 */
            MCPWM_InitStructure.FAIL0_Signal_Sel = FAIL_SEL_CMP;      /* FAIL0事件信号选择，比较器或IO口 */
            MCPWM_InitStructure.FAIL0_Polarity = HIGH_LEVEL_ACTIVE;   /* FAIL0事件极性选择，高有效 */

            MCPWM_InitStructure.FAIL1_INT_EN = ENABLE;                /* FAIL1事件 中断使能或关闭 */
            MCPWM_InitStructure.FAIL1_INPUT_EN = ENABLE;              /* FAIL1通道急停功能打开或关闭 */
            MCPWM_InitStructure.FAIL1_Signal_Sel = FAIL_SEL_CMP;      /* FAIL1事件信号选择，比较器或IO口 */
            MCPWM_InitStructure.FAIL1_Polarity = HIGH_LEVEL_ACTIVE;   /* FAIL1事件极性选择，高有效或低有效 */ 
        
            MCPWM_Init(&MCPWM_InitStructure);
            
            mIPD_CtrProc.hDriverPolarity = MCPWM_IO01;                /* 读出驱动极性 */
        }
      ```
---