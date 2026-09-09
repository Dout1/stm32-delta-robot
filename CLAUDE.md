CLAUDE.md – 工创赛智能分拣 Delta 机器人电控项目

项目概述

本项目为全国大学生工程训练综合能力竞赛（工创赛）智能分拣赛道参赛作品。采用 Delta 并联机器人 + 三指柔性夹爪方案，主控芯片为 STM32F407IGT6（野火骄阳开发板），实现物料的分拣、搬运与分类。

校赛时间：2026年10月中旬  
电控负责人：[陆华昌]  
当前阶段：硬件平台搭建完成，USART1 调试串口已打通，正在进行逆运动学算法开发。

技术栈

• MCU：STM32F407IGT6 (Cortex-M4, FPU, 168MHz)

• 开发环境：Keil MDK v5.36 (Compiler V5.06 update 7)

• 底层驱动：STM32CubeMX + HAL库 (F4 v1.28.0) 或者 v1.28.3 

• 工程架构：分层模块化（Core/BSP/Algorithm/Module/App/Utilities/Docs）

• 版本管理：Git (建议)

硬件配置

主控板

• 型号：野火骄阳 STM32F407IGT6

• 板载晶振：25MHz HSE

• 板载调试器：无。骄阳板没有板载 ST-Link，烧录/仿真需外接调试器到 1×5P XH2.54 的 SWD 座。

• 板载 USB 转串口：CH340（对应 USART1，**PB6/TX、PB7/RX**，经 J6/J7 跳线帽连通；跳线帽未装则不通）

时钟树（已配置验证）

时钟域 频率 来源

HSE 25 MHz 外部晶振

PLL (SYSCLK) 168 MHz HSE→PLL (M=25, N=336, P=2)

APB1 42 MHz SYSCLK / 4

APB2 84 MHz SYSCLK / 2

PLLQ 7 (忽略) 未使用

引脚分配（已确认）

步进电机驱动（TIM PWM）

轴 脉冲(PUL) 方向(DIR) 使能(ENA) 备注

轴1 PE5 (TIM9_CH1) PE1 PE0 低电平使能

轴2 PB9 (TIM4_CH4) PB8 PE4 低电平使能

轴3 PH6 (TIM12_CH1) PD7 PD6 PH0/PH1被HSE占用，DIR/ENA改用PD6/PD7

PWM参数：PSC=83, ARR=999, Pulse=499 → 1kHz, 50% duty

串口

用途 串口 TX RX 波特率 状态

调试（PC） USART1 **PB6** **PB7** 115200 🔧 已改引脚，待实测（原 PA9/PA10 在骄阳板上未连到 CH340）

视觉板 USART2 PD5 PD6 115200 🔧 待配置

串口屏 UART5 或 USART6 PC12 PD2 9600-115200 ❌ 待采购后配置

其他

• LED_USER：PA15（低电平点亮）

• 板载额外LED：PE2, PG15, PB8

• SYS Timebase：TIM5（避免与TIM2/TIM4冲突）

驱动器与执行器

• 步进电机驱动器：TB6600 × 3（推荐，支持3.3V逻辑电平直连）

• 步进电机：42步进电机

• 三指柔性夹爪：42步进电机

• 串口屏：推荐淘晶驰或迪文3.5寸（待采购）

软件架构


DeltaRobot_F407/
├── Core/                 # HAL库核心文件（自动生成）
│   ├── Inc/
│   └── Src/
├── BSP/                  # 板级支持包（板载外设驱动）
│   ├── bsp_led.c/h
│   ├── bsp_key.c/h
│   └── bsp_uart.c/h      # 串口初始化封装
├── Algorithm/            # 算法模块
│   ├── algo_delta_kinematics.c/h  # Delta逆运动学（待实现）
│   ├── algo_trapezoid.c/h         # 梯形加减速（待实现）
│   └── algo_vision_protocol.c/h   # 视觉协议解析（待实现）
├── Module/               # 功能模块
│   ├── mod_stepper.c/h           # 步进电机控制（PWM+方向+使能）
│   ├── mod_gripper.c/h           # 夹爪控制
│   └── mod_display.c/h           # 串口屏显示
├── App/                  # 应用层
│   ├── app_main.c/h             # 主状态机
│   └── app_task.c/h             # 任务调度（非RTOS）
├── Utilities/            # 工具函数
│   ├── util_debug.c/h           # 调试打印封装
│   └── util_fifo.c/h            # FIFO缓冲区
└── Docs/                 # 文档
    └── pin_map.xlsx


开发进度（截至2026-09-06）

✅ 已完成

1. 时钟树配置：168MHz SYSCLK，已验证
2. GPIO点灯：PA15 LED点亮成功
3. TIM PWM配置：三轴步进脉冲引脚已分配，参数1kHz/50%
4. USART1调试串口：115200 8N1，中断收发，printf重定向，串口助手测试通过
5. NVIC优先级分组：Group 2（2位抢占+2位子优先级）
   • SysTick: Preemption=3, Sub=0

   • TIM2 (HAL时基): Preemption=2, Sub=0

   • USART1: Preemption=3, Sub=0

6. 工程架构搭建：分层模块化目录结构，空文件已约定接口

🔧 进行中

• USART2配置（预留给视觉板）

• 逆运动学算法编写（algo_delta_kinematics.c）

❌ 待办

• Delta机械尺寸参数（杆长、基座半径、末端行程）

• 视觉板通信协议（帧头帧尾、数据格式、校验方式）

• 串口屏采购与UI设计

• 梯形加减速算法

• 限位开关/原点回归逻辑

• 整机联调（视觉→串口→逆解→PWM→电机→夹爪）

开发规则

1. 所有用户代码必须写在 USER CODE BEGIN/END 区域内，避免重新生成代码时被覆盖。
2. 禁止修改自动生成的 .c/.h 文件（如 main.c 的自动生成部分、stm32f4xx_hal_msp.c 等）。
3. 新增模块统一放在 Module/ 或 Algorithm/ 目录，不要在 Core/ 下添加自定义文件。
4. 中断优先级分配原则：
   • SysTick 抢占优先级最高（0 或 3，视分组而定）

   • 用户中断（TIM、USART）抢占优先级 > 0，避免阻塞 HAL_Delay

   • 视觉串口（USART2）优先级应高于调试串口（USART1）

5. HAL库串口接收为一次性机制：每次回调后必须重新调用 HAL_UART_Receive_IT()。
6. 步进电机使能（ENA）低电平有效，初始化时置高防止误动作。
7. 所有全局变量命名加前缀：g_（全局）、s_（静态）、u_（用户模块内部）。
8. 函数命名规范：模块名_动作_对象()，如 mod_stepper_set_speed(axis, speed)。

注意事项

硬件

• 骄阳板有两个 Type-C 口：USB Device（STM32 内置 USB，PA11/PA12）和 USB 转串口（CH340，对应 USART1 的 PB6/PB7）。插错口设备管理器不会出现 COM 口，调试时务必分清。USART1 走的是 J6/J7 跳线帽，必须装上。

• TB6600驱动器需要独立12-24V供电，并与骄阳板共地。

• 串口屏建议选购5V供电型，注意与3.3V逻辑的电平匹配（多数串口屏支持3.3V）。

• 板载串口屏接口与EBF接口共用，引脚为PD2/PC12（对应UART5或USART6）。

软件

• 编译前确保Keil中勾选了 Use MicroLIB（用于printf重定向）。

• 如需使用FreeRTOS，需将HAL时基从TIM5改为非SysTick定时器（如TIM6/7），且SysTick优先级设为15。

• 逆运动学算法使用浮点运算（STM32F407有硬件FPU，编译时需使能FPU选项）。

比赛规则（摘自资料库）

• 决赛需提交任务命题文档，内容包括场景规划、物料尺寸/形状/颜色/重量等。文档不得雷同，不得出现校名/队名/特殊标记。

• 作品创意设计评分包括创新性、美观性、结构合理性。

• 分拣装置上需安装高亮显示屏（仅显示功能），显示序号、货物名称、图片、分拣数量等信息。因此必须使用串口屏。

参考资料

1. 《野火STM32 HAL库开发实战指南》（F4系列）
2. 《STM32F4xx中文参考手册》
3. 野火骄阳开发板原理图（PDF）
4. 工创赛智能分拣赛道命题文档模板
5. GitHub开源项目：SergTyapkin/Delta-4-DOF-control-system、SickSad/DeltaRobot

联系方式

• 电控负责人：[姓名]

• 机械负责人：[姓名]

• 视觉负责人：[姓名]

• 指导老师：[姓名]

本文档由 Claude 于 2026-09-06 生成，请根据实际进度定期更新。