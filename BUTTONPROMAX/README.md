# BUTTONPROMAX

第二次作业 · 基础任务 1：**定时器中断消抖按键（非阻塞）**
STM32F103C8T6 + STM32CubeMX 6.18.1 + Keil MDK-ARM（ARM Compiler 6.24）

## 功能

| 操作 | 现象 |
| --- | --- |
| 上电 | LED 常灭 |
| 短按（< 1s） | 常亮 / 常灭 之间切换 |
| 长按 ≥ 1s 后松手 | 进入 5Hz 快闪（亮 100ms、灭 100ms） |
| 快闪时再次长按 ≥ 1s 松手 | 退出快闪，回到进入前的常亮 / 常灭 |
| 快闪时短按 | 退出快闪并切换一次常亮 / 常灭 |

## 程序骨架（讲义第 6.5 / 7.5 节）

```
TIM2 每 1ms 中断：采样 PA9 → 连续 20 次为按下才确认 → 置 key_pressed / key_released
                         ↓ 事件标志（volatile）
主循环 while(1)：按下事件记时间 → 松开事件算差值 → 短按 / 长按 → 改模式
                         ↓
LED：常亮 / 常灭（写电平）、快闪（每 100ms 翻转，看时间不阻塞）
```

中断里只留记号，判断和输出都在主循环；全程没有一个 `HAL_Delay`。

## 硬件接线

- 按键：一端接 **PA9**，另一端接 **GND**；PA9 内部上拉（CubeMX 中 Pull-up）。
- LED：板载 LED 接在 3.3V 与 **PC13** 之间，**低电平点亮**。
- ST-Link：3.3V、GND、SWDIO→PA13、SWCLK→PA14。

## 关键参数

| 项目 | 数值 |
| --- | --- |
| SYSCLK / APB1 | 8MHz（HSI） |
| TIM2 PSC / ARR | 7 / 999 → 1MHz 计数、**1ms** 更新中断 |
| 消抖 | 连续 20 次采样一致 = **20ms** |
| 长按阈值 | `HAL_GetTick()` 差值 ≥ **1000ms** |
| 快闪 | 每 **100ms** 翻转一次，周期 200ms = **5Hz** |

时基公式：

```
f_cnt = 8MHz / (PSC + 1) = 8MHz / 8 = 1MHz
T     = (ARR + 1) / f_cnt = 1000 / 1MHz = 1ms
```

## 代码位置（Core/Src/main.c 的 USER CODE 区）

- `KEY_DEBOUNCE_MS` / `KEY_LONG_MS` / `BLINK_HALF_MS`：三个时间常数
- `volatile key_cnt / key_pressed / key_released`：中断与主循环共享的事件变量
- `HAL_TIM_PeriodElapsedCallback()`：1ms 采样 + 20 次消抖 + 报告事件
- `while (1)`：按下记时间、松开算差值、短按/长按切换、快闪翻转

## 编译

1. 打开 `MDK-ARM/BUTTONPROMAX.uvprojx`。
2. 已设置 ARM Compiler **V6.24**；若用 CubeMX 重新生成后提示 “Compiler Version 5 not available”，
   在 *Options for Target → Target → ARM Compiler* 里改回 V6.24。
3. F7 编译（0 Error / 0 Warning），F8 下载。
4. 产物：`MDK-ARM/BUTTONPROMAX/BUTTONPROMAX.hex`。

## 目录

```
BUTTONPROMAX/
├─ BUTTONPROMAX.ioc          CubeMX 工程
├─ Core/Inc, Core/Src        应用代码（main.c 等）
├─ Drivers/                  HAL + CMSIS
└─ MDK-ARM/                  Keil 工程与编译产物
```
