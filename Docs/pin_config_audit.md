# CLAUDE.md 与 CubeMX(.ioc) 配置对照检查报告

- 检查对象：`CLAUDE.md`（v2026-09-06）、`DeltaRobot_F407.ioc`（CubeMX 6.18.1）、`Core/` 生成代码、`MDK-ARM/DeltaRobot_F407.uvprojx`
- 检查日期：2026-09-07
- 结论：**发现 5 处严重不一致（其中 3 处会导致电机完全不动）+ 6 处中等 + 4 处轻微**

---

## 一、工程任务（据 CLAUDE.md）

| 项目 | 内容 |
|------|------|
| 赛事 | 全国大学生工程训练综合能力竞赛（工创赛）智能分拣赛道 |
| 目标 | Delta 并联机器人 + 三指柔性夹爪，完成物料分拣、搬运、分类 |
| 主控 | STM32F407IGT6（野火骄阳开发板，25MHz HSE） |
| 执行器 | TB6600 驱动器 ×3 + 42 步进电机 ×3；夹爪 42 步进电机 ×1 |
| 通信 | USART1 调试、USART2 视觉板、UART5/USART6 串口屏 |
| 当前阶段 | 硬件搭建完成、USART1 已调通，正在做逆运动学 |
| 下一步 | 逆解算法 → 梯形加减速 → 视觉协议 → 串口屏 → 整机联调 |

技术栈：Keil MDK + ARMCC V5.06 update 7、HAL FW_F4 V1.28.3、分层模块化（Core/BSP/Algorithm/Module/App）。

---

## 二、严重不一致（必须处理，否则功能失效）

### S1. 三轴脉冲引脚与文档完全不符 ★★★

| 轴 | CLAUDE.md 写的 | .ioc 实际配置 | 结果 |
|----|---------------|--------------|------|
| 轴1 | PE5 (TIM9_CH1) | **PI5 (TIM8_CH1)** | 不符 |
| 轴2 | PB9 (TIM4_CH4) | **PI6 (TIM8_CH2)** | 不符 |
| 轴3 | PH6 (TIM12_CH1) | **PI7 (TIM8_CH3)** | 不符 |
| （无） | — | **PC9 (TIM8_CH4)** | 文档未记录 |

`Core/Src/tim.c` 中 `HAL_TIM_MspPostInit()` 确认：PC9 / PI5 / PI6 / PI7 均复用 AF3 挂 TIM8。文档里的 PE5/PB9/PH6 一个都没被用到。

### S2. TIM8 CH1~CH3 配成了"输出比较-冻结"，不产生波形 ★★★

`tim.c` 第 75~99 行：

```c
sConfigOC.OCMode = TIM_OCMODE_TIMING;   /* CH1/CH2/CH3 —— 冻结模式，引脚不翻转 */
sConfigOC.Pulse  = 499;
...
sConfigOC.OCMode = TIM_OCMODE_PWM1;     /* CH4 */
sConfigOC.Pulse  = 0;                   /* 占空比 0%，恒低电平 */
```

对应 .ioc：`SH.S_TIM8_CH1.0=TIM8_CH1,Output Compare1 CH1`（不是 PWM Generation）。
**结论：4 路通道目前没有任何一路能输出脉冲，电机一个脉冲都收不到。**

### S3. main.c 从未启动 PWM/OC 输出 ★★★

`main.c` 只调用了 `MX_TIM8_Init()`，全工程没有 `HAL_TIM_PWM_Start()` / `HAL_TIM_OC_Start()`。即使模式改对，不调用 Start 也没有输出。

### S4. PWM 实际频率是 2kHz，不是文档写的 1kHz ★★

- TIM8 挂载在 **APB2**，`.ioc` 中 `RCC.APB2TimFreq_Value = 168000000`（APB2=84MHz，预分频≠1 故定时器时钟 ×2）
- 实际频率 = 168MHz / (83+1) / (999+1) = **2000 Hz**
- 文档"PSC=83, ARR=999 → 1kHz"只在定时器时钟为 84MHz 时成立（即 TIM4/TIM12 等 APB1 定时器）
- 若要在 TIM8 上得到 1kHz：PSC 改 **167**（ARR 保持 999），或 ARR 改 1999（占空比相应改为 999）

### S5. 轴3 的 DIR/ENA 引脚文档错误，且文档自身冲突 ★★

- CLAUDE.md 引脚表：轴3 ENA=PD6、DIR=PD7
- .ioc 实际：AXIS3_ENA=**PI10**、AXIS3_DIR=**PI11**
- 更关键：CLAUDE.md 的串口表又把 **PD6 分配给 USART2_RX** —— PD6 同时被轴3 ENA 和 USART2 占用，文档内部自相矛盾
- 且"PH0/PH1 被 HSE 占用，故 DIR/ENA 改用 PD6/PD7"这个推导本身站不住脚（HSE 占用 PH0/PH1 与 DIR/ENA 选哪个脚没有因果关系）

---

## 三、中等不一致

### M1. HAL 时基定时器：文档 TIM5，实际 TIM2
- `.ioc`：`NVIC.TimeBase=TIM2_IRQn`、`VP_SYS_VS_tim2.Mode=TIM2`，`stm32f4xx_hal_timebase_tim.c` 也是 TIM2
- 文档自相矛盾：硬件配置段写"TIM5（避免与 TIM2/TIM4 冲突）"，进度段又写"TIM2 (HAL时基): 2/0"。**实际是 TIM2**
- 影响：文档"避免与 TIM2 冲突"的选型理由失效；后续若上 FreeRTOS 需按 TIM2 重新规划

### M2. 存在第 4 轴，文档完全未记录
`.ioc` 有 `AXIS4_ENA=PF1`、`AXIS4_DIR=PF2`，配合 TIM8_CH4(PC9)。推测是留给夹爪的，但文档引脚表只有 3 轴，需明确。

### M3. 板载 LED 描述与配置冲突
- 文档："板载额外 LED：PE2, PG15, PB8"
- 实际：`PB8` 已被占用为 **AXIS2_DIR**（与 LED 用途冲突）；PE2/PG15 在 .ioc 中根本没配置

### M4. 串口屏"UART5 或 USART6"的说法有误
- PC12 在 F407 上**只能**做 UART5_TX（AF8），USART6 的引脚是 PC6/PC7 或 PG14/PG9，PC12 与 USART6 无关
- 应把文档改为**固定 UART5：PC12(TX) / PD2(RX)**，"USART6"这个备选项删掉
- PD5/PD6 作为 USART2（视觉板）是正确的（AF7），保留

### M5. SysTick 优先级并未"最高"
- .ioc：SysTick = 3/0，USART1 = 3/0，TIM2 = 2/0
- 文档规则写"SysTick 抢占优先级最高"，实际 SysTick 与 USART1 同级，且真正的时基 TIM2(2) 比 SysTick(3) 更高
- 影响有限（时基在 TIM2 上，HAL_Delay 正常），但文档规则描述需修正

### M6. 需查原理图确认（无法从配置文件判断）
PI5/PI6/PI7/PI10/PI11 与 PC9 是否真的引出到骄阳板排针、是否与板载 FMC SDRAM / LCD 复用（F407 的 GPIOI 多为 FMC 复用脚）。**建议在改板前用原理图确认一遍。**

---

## 四、轻微 / 文档笔误

| # | 位置 | 文档 | 实际 | 说明 |
|---|------|------|------|------|
| L1 | 时钟树 | PLLQ = 7 | **PLLQ = 4**（main.c PLL.PLLQ=4） | 无关紧要但应改 |
| L2 | 开发环境 | Keil MDK v5.36 | .ioc 写 MDK-ARM V5.32 | 无实质影响 |
| L3 | 工具链 | — | .ioc `CompilerLinker=GCC` 为残留字段，uvprojx 实为 `V5.06 update 7 :: .\ARMCC` | 忽略 |
| L4 | 责任人 | 抬头"电控负责人：[陆华昌]"，联系方式段仍是"[姓名]"占位符 | — | 待补 |

---

## 五、核对一致、没有问题的部分 ✅

| 项 | 文档 | .ioc/代码 | 结论 |
|----|------|-----------|------|
| MCU | STM32F407IGT6 | `Mcu.CPN=STM32F407IGT6`, LQFP176 | ✅ |
| HSE | 25MHz | `RCC.HSE_VALUE=25000000`，PH0/PH1 为 OSC | ✅ |
| PLL | M=25 N=336 P=2 → 168MHz | 完全一致，SYSCLK=168MHz | ✅ |
| APB1/APB2 | 42 / 84 MHz | 完全一致 | ✅ |
| USART1 | PA9/PA10, 115200, 8N1, 中断 | `usart.c` 全部一致 | ✅ |
| NVIC 分组 | Group 2 | `NVIC_PRIORITYGROUP_2` | ✅ |
| NVIC 优先级 | SysTick 3/0、TIM2 2/0、USART1 3/0 | 完全一致 | ✅ |
| LED_USER | PA15 低电平点亮 | PA15 输出，PinState=SET（初始熄灭） | ✅ |
| 轴1 ENA/DIR | PE0 / PE1 | AXIS1_ENA=PE0, AXIS1_DIR=PE1 | ✅ |
| 轴2 ENA/DIR | PE4 / PB8 | AXIS2_ENA=PE4, AXIS2_DIR=PB8 | ✅ |
| ENA 初始电平 | 高（防误动作） | 4 路 ENA 均 `GPIO_PIN_SET` | ✅ 符合规则6 |
| PH0/PH1 | 被 HSE 占用 | 确认被占 | ✅ |
| USART2/UART5/USART6 | 待配置 | 均未配置 | ✅ 与"待办"一致 |
| 编译器 & FPU | ARMCC V5.06u7、FPU 使能、MicroLIB | `pCCUsed=V5.06 update 7`、`useUlib=1`、`Cpu` 含 FPU2 | ✅ |
| HAL 版本 | v1.28.0 或 1.28.3 | `FW_F4 V1.28.3` | ✅ |

---

## 六、建议修正方案

### 方案 A：以现有 .ioc 为准，改文档 + 补代码（推荐）

理由：三轴 + 夹爪共 4 路脉冲用**同一个 TIM8 的 4 个通道**，时钟同源、便于同步，比文档里 TIM9/TIM4/TIM12 三个不同总线定时器（时钟分别是 168/84/84 MHz，PSC 得各算一遍）更靠谱。

需要做的：

1. **CubeMX**：TIM8 的 CH1/CH2/CH3 由 `Output Compare` 改为 **`PWM Generation CHx`**；CH4 若用作夹爪也改为 PWM Generation；Pulse 全部 499
2. **CubeMX**：TIM8 Prescaler 由 83 改为 **167**（APB2 定时器时钟 168MHz → 1kHz）
3. **代码**：在 `main.c` 的 `USER CODE BEGIN 2` 中启动输出
   ```c
   HAL_TIM_PWM_Start(&htim8, TIM_CHANNEL_1);
   HAL_TIM_PWM_Start(&htim8, TIM_CHANNEL_2);
   HAL_TIM_PWM_Start(&htim8, TIM_CHANNEL_3);
   HAL_TIM_PWM_Start(&htim8, TIM_CHANNEL_4);
   ```
   （TIM8 是高级定时器，MOE 位由 `HAL_TIM_PWM_Start` 自动打开，无需手动处理）
4. **文档**：更新引脚表为下表

### 方案 B：以文档为准，改 .ioc

把 TIM8 换成 TIM9_CH1(PE5) + TIM4_CH4(PB9) + TIM12_CH1(PH6)。**不推荐**：三个定时器分属 APB2/APB1/APB1，时钟源不同（168/84/84 MHz），同参数下频率差 2 倍，梯形加减速会很难对齐。

### 修正后的引脚表（建议写入 CLAUDE.md）

| 轴 | 脉冲 PUL | 方向 DIR | 使能 ENA | 备注 |
|----|---------|---------|---------|------|
| 轴1 | PI5 (TIM8_CH1) | PE1 | PE0 | ENA 低有效，初始 SET |
| 轴2 | PI6 (TIM8_CH2) | PB8 | PE4 | ENA 低有效，初始 SET |
| 轴3 | PI7 (TIM8_CH3) | PI11 | PI10 | ENA 低有效，初始 SET |
| 轴4/夹爪 | PC9 (TIM8_CH4) | PF2 | PF1 | ENA 低有效，初始 SET |

PWM：TIM8，定时器时钟 168MHz，**PSC=167, ARR=999, Pulse=499 → 1kHz / 50%**

| 串口 | TX | RX | 波特率 | 状态 |
|------|----|----|--------|------|
| 调试 | PA9 | PA10 | 115200 | ✅ 已通 |
| 视觉板 | PD5 | PD6 | 115200 | 待配置 |
| 串口屏 | PC12 (UART5_TX) | PD2 (UART5_RX) | 待定 | 待采购（**删除 USART6 选项**） |

其他：LED_USER = PA15（低电平点亮）；HAL 时基 = **TIM2**（非 TIM5），抢占 2/0。

---

## 七、上电前自检清单

- [ ] 原理图确认 PI5/PI6/PI7/PC9、PI10/PI11、PE0/PE1/PE4、PF1/PF2 已引出且无板载复用
- [ ] CubeMX：TIM8 CH1~CH4 全部为 PWM Generation
- [ ] CubeMX：TIM8 PSC=167
- [ ] main.c：4 路 `HAL_TIM_PWM_Start`
- [ ] 用示波器/万用表量 PI5 对地，确认 1kHz 方波
- [ ] 更新 CLAUDE.md 引脚表、串口表、时钟树
