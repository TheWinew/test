# 固件架构

## 目标

这套分层优先隔离三类变化：

1. 产品控制流程变化：解锁、失联保护、校准、配平和偏航稳定。
2. 设备能力变化：CRSF、IMU、LED、按键、Flash、舵机和扑翼机构。
3. 板级接线变化：GPIO、TIM、通道、时钟树和复用功能。

依赖方向固定为：

```text
Core/main
    -> Application/FlightApp
        -> Hardware/Butterfly -> Hardware/Servo -> Board/BoardServoPwm
        -> Hardware/Crsf, Key, LED, LSM6DSO, SettingsStorage
        -> System/Delay
            -> STM32F4 StdPeriph + CMSIS
```

上层可以调用下层；下层不得读取或修改上层状态。

## 各层责任

| 目录 | 责任 | 不允许出现 |
|---|---|---|
| `Core` | 复位入口、中断入口、进入产品应用 | 控制策略、设备初始化清单 |
| `Application` | 产品生命周期、模式切换、安全策略、模块编排 | GPIO 引脚、TIM 通道、寄存器 |
| `Hardware` | 一个设备或机构的稳定 API | 产品解锁规则、遥控通道用途 |
| `Board` | MCU 与 PCB 的具体接线、时钟和外设映射 | 扑翼、油门、配平等业务概念 |
| `System` | 与产品无关的基础服务 | 设备和业务策略 |

## 当前关键边界

舵机输出调用链为：

```text
FlightApp
  -> Butterfly_Set* / Butterfly_Update
  -> Servo_SetBothUs
  -> BoardServoPwm_Write
  -> TIM4_CH3 / TIM4_CH4
```

因此：

- 修改扑翼波形，只改 `Hardware/Butterfly.c`。
- 修改舵机合法脉宽，只改 `Hardware/Servo.h/.c`。
- PB8/PB9 改到其他引脚，或 TIM4 改成其他定时器，只改 `Board/BoardServoPwm.c`。
- 修改解锁、失联、校准或遥控通道含义，改 `Application/FlightApp.c`；修改通用 PID 算法，改 `Application/PID.c`，参数和启停条件仍由 `FlightApp.c` 所有。

当前只有一个 STM32F4 板级实现，所以没有引入 `base + ops`。当同一固件需要在两种 MCU、两套 PWM 后端之间运行，或需要 PC 单元测试替身时，再把 `BoardServoPwm` 升级成 `config + ops`；现在引入函数指针只会增加理解成本。

## 状态所有权

- 应用运行状态为 `FlightApp.c` 的 file-local `static` 数据，只能被应用逻辑修改。
- CRSF 中断缓冲和统计由 `Crsf.c` 所有。
- IMU 零偏缓存由 `LSM6DSO.c` 所有，通过函数访问，不再借用应用全局变量。
- 只读硬件映射应留在 `Board` 文件内。

禁止新增跨模块的裸 `extern` 状态。跨模块协作使用函数和显式参数。

## 初始化与循环

```text
main
  -> FlightApp_Run
      -> LED / Key / Butterfly / IMU / CRSF 初始化
      -> while (1)
          -> 输入采集
          -> 安全与模式判断
          -> 控制量计算
          -> 执行器更新
          -> 状态指示
          -> 固定周期延时
```

`main.c` 不再维护模块清单之外的产品细节。若后续模块继续增加，应先把 `FlightApp_Run` 拆成 `FlightApp_Init` 与 `FlightApp_Step`，然后再按真实变化压力拆出 `FlightControl`、`Calibration` 和 `Failsafe`，不要仅为了目录好看而拆文件。

## 工程红线

1. `Application` 中不得出现 `GPIO_Pin_*`、`TIMx`、`RCC_*` 或直接寄存器访问。
2. 所有公开 API 检查空指针、范围和初始化状态；错误不得静默伪装成成功。
3. 中断只搬运数据和记录事件，不执行 Flash 擦写、延时或控制策略。
4. 失联、低电压和传感器连续读失败必须收敛到安全输出。
5. 修改时钟树后，必须重新验证 PWM 的 1 us 计数基准和 20 ms 周期。
6. PID 必须使用实际循环周期更新；P/I/D 计算、输出限幅和 anti-windup 不得重新散落到 `FlightApp.c`。
