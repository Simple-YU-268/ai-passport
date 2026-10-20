# FoloToy AI Passport 完全逆向工程报告

> 分析日期: 2026-10-19  
> 分析仓库: https://github.com/FoloToy/ai-passport  
> 分支: main, demo/claude-buddy-port  
> 分析工具: REA (Reverse Engineering Anything) v6.7.0

---

## Phase 1: 侦察 — 硬件与架构全貌

### 硬件规格

| 项目 | 详情 |
|------|------|
| **MCU** | ESP32-C3 (RISC-V 32位, 160MHz) |
| **Flash** | 8MB (无 PSRAM) |
| **显示屏** | ST7789P3, 240×320, RGB565, SPI 80MHz |
| **音频** | ES8311 编解码器, I²S 全双工 |
| **电池** | CW2017 电量计, 500mAh |
| **按键** | 3键 ADC 阶梯 (GPIO0/ADC1_CH0) |
| **无线** | 2.4GHz Wi-Fi + BLE 5.0 (NimBLE) |
| **NFC** | 被动 NTAG213 标签 |
| **接口** | USB-C (Serial/JTAG, GPIO18/19) |

### 引脚映射 (bsp_pins.h)

| GPIO | 功能 | 备注 |
|------|------|------|
| 0 | 按键 ADC | ADC1_CH0, 三键共用 |
| 1 | LCD CS | SPI |
| 2 | I2S DOUT | → ES8311 |
| 3 | I2S WS | |
| 4 | I2S DIN | ← ES8311 |
| 5 | I2S BCLK | |
| 6 | I2S MCLK | |
| 7 | I2C SCL | 共享总线 |
| 8 | LCD SCLK | SPI |
| 9 | LCD MOSI | 无 MISO |
| 10 | I2C SDA | |
| 18/19 | USB | Serial/JTAG |
| 20 | LCD DC | |
| 21 | 背光 PWM | LEDC |

---

## Phase 2: 静态分析 — 源码架构与模块

### 分区表 (partitions.csv)

```
nvs,      data, nvs,     0x9000,   0x6000
phy_init, data, phy,     0xf000,   0x1000
factory,  app,  factory, 0x10000,  0x7f0000
```

### SDK 配置关键项

- ESP-IDF 5.5.3
- LVGL 9.5.0 (esp_lvgl_port 2.9.0)
- NimBLE 单连接 peripheral
- USB Serial/JTAG 控制台 (非 UART0)
- 无 PSRAM, DMA 用内部 RAM

### 依赖组件

| 组件 | 版本 |
|------|------|
| espressif/button | 4.2.0 |
| espressif/esp_codec_dev | 1.6.2 |
| espressif/esp_lvgl_port | 2.9.0 |
| lvgl/lvgl | 9.5.0 |

### 代码结构 (4841 行 main)

**main/ (应用层)**
- `main.c` (279行) — 入口, 初始化序列, 菜单生命周期
- `demo_display.c` — 显示测试 (色块+背光)
- `demo_button.c` — 按键+ADC 电压显示
- `demo_audio.c` — 1kHz 方波 + 录音回放
- `demo_battery.c` — 电量/电压显示
- `demo_wifi.c` — Wi-Fi 扫描
- `demo_ble.c` — BLE 广播 (设备名 "FoloPassport")
- `demo_low_power.c` — 浅睡/深睡测试
- `demo_navigation.c` — 导航状态机
- `ui_pixel.c` — 像素风 UI 组件

**components/bsp/ (板级支持包)**
- `bsp_display.c` — ST7789P3 初始化序列 (厂商专属)
- `bsp_display_lvgl.c` — LVGL 接入, 圆角遮罩
- `bsp_display_rounding.c` — 圆角算法
- `bsp_audio.c` — ES8311 驱动 (570行, 最复杂)
- `bsp_battery.c` — CW2017 驱动, 自定义电池 profile
- `bsp_button.c` — ADC 按键解码
- `bsp_i2c.c` — 共享 I2C 总线管理

---

## Phase 3: 运行时分析 — 通信协议与行为

### BLE 协议 (demo_ble.c)

- **设备名**: `FoloPassport`
- **模式**: 不可连接广播 (Non-connectable advertising)
- **Flags**: DISC_GEN + BREDR_UNSUP
- **状态机**: OFF → STARTING → ADVERTISING → FAILED

### Wi-Fi 行为 (demo_wifi.c)

- 仅扫描, 不连接
- 存储: `WIFI_STORAGE_RAM` (不持久化)
- 结果缓存: 5 个网络

### 音频子系统 (bsp_audio.c)

关键发现:
- **采样率**: 默认 16kHz, 16bit, 单声道
- **音量控制**: 0-100% 软件音量
- **休眠序列**: 自定义 ES8311 寄存器序列 (15 条指令)
- **已知 bug 绕过**: esp_codec_dev 1.6.2 丢弃错误 → 用包装层追踪

### 电池管理 (bsp_battery.c)

- **电量计**: CW2017 @ I2C 0x63
- **自定义 Profile**: 80 字节, 已硬编码
- **休眠**: CONFIG 寄存器写 0xF0
- **电压计算**: raw × 312.5 = 微伏

### 按键系统 (bsp_button.c)

- **ADC 阶梯**: 三键共用 GPIO0
- **电压窗口**: 半开区间防边界冲突
- **采样**: 1000ms 共享读取, 平均滤波
- **事件**: PRESS, CLICK, DOUBLE, LONG, RELEASE

### 电源管理 (demo_low_power.c)

- **浅睡**: 2 秒定时器唤醒, 保持外设状态
- **深睡**: 5 秒定时器 + GPIO0 按键唤醒
- **深睡准备序列**: CW2017 → ES8311 → I2S → I2C → LCD (严格顺序)
- **唤醒守卫**: key_guard 状态机防误触发

---

## Phase 4: 深度逆向 — 关键算法与逻辑

### ST7789P3 初始化序列

```
PORCTRL (0xB2): 05 05 00 33 33
GCTRL   (0xB7): 35
VCOMS   (0xBB): 21
LCMCTRL (0xC0): 2C
VDVVRHEN(0xC2): 01
VRHS    (0xC3): 0B
VDVSET  (0xC4): 20
FRCTRL2 (0xC6): 0F (60Hz)
PWCTRL1 (0xD0): A7 A1 → A4 A1 (重发覆盖)
D6      (0xD6): A1
PVGAMCTRL(0xE0): D0 04 08 0A 09 05 2D 43 49 09 16 15 26 2B
NVGAMCTRL(0xE1): D0 03 09 0A 0A 06 2E 44 40 3A 15 15 26 2A
```

### 圆角遮罩算法 (bsp_display_rounding.c)

- 半径: 30px (BSP_LVGL_SCREEN_RADIUS)
- 逐行计算可见区间, 外部填充黑色
- 优化: 每边缘行最多 30 次迭代

### 按键守卫状态机 (key_guard.h)

```
ARMED → (release 事件 | 超时 | 物理电平已释放) → DISARMED
```

### 导航状态机 (demo_navigation.c)

```
状态: selected, active, count
输入: UP/DOWN/OK_CLICK/OK_LONG
输出: REFRESH/ENTER/EXIT/FORWARD
```

---

## Phase 5: 安全评估与定制建议

### 安全发现

| 级别 | 发现 |
|------|------|
| **低** | BLE 广播不含敏感数据, 但可被追踪 |
| **低** | Wi-Fi 密码不持久化 (RAM only) |
| **中** | 深睡唤醒后按键守卫有竞态窗口 (250ms) |
| **中** | I2C 扫描若用临时总线会导致总线死锁 (已注释警告) |
| **低** | NVS 不自动擦除, 保留用户数据 |

### 定制建议

1. **固件开发**: 用 ESP-IDF 5.5.3 + 此 BSP 作为基线
2. **MicroPython**: 可用 ai-passport-micropython 仓库快速原型
3. **AI Agent 集成**: claude-buddy-port 分支有完整 BLE NUS 协议实现
4. **低功耗优化**: 参考 es8311-low-power-suspend 分支
5. **电池校准**: 修改 `s_battery_profile[80]` 数组适配不同电芯

### 关键文件速查

| 需求 | 文件 |
|------|------|
| 改引脚 | `components/bsp/include/bsp_pins.h` |
| 改显示屏 | `components/bsp/src/bsp_display.c` |
| 改音频参数 | `components/bsp/src/bsp_audio.c` |
| 改电池 profile | `components/bsp/src/bsp_battery.c` |
| 加新 demo 页 | `main/demo.h` + `main/demo_*.c` |
| 改 LVGL 配置 | `sdkconfig.defaults` |
| 改分区表 | `partitions.csv` |

---

## 附录: claude-buddy-port 分支分析

该分支实现了完整的 AI 伴侣功能:

### BLE NUS 协议

- **Service UUID**: 9e ca dc 24 0e e5 a9 e0 93 f3 a3 b5 01 00 40 6e
- **RX UUID**:    9e ca dc 24 0e e5 a9 e0 93 f3 a3 b5 02 00 40 6e
- **TX UUID**:    9e ca dc 24 0e e5 a9 e0 93 f3 a3 b5 03 00 40 6e
- **MTU**: 244 bytes max fragment

### JSON 命令协议

```json
// 心跳
{"total": 5, "running": 1, "waiting": 3, "tokens": 12345, "tokens_today": 67890, "msg": "Working...", "entries": ["task1", "task2"], "prompt": {"id": "abc", "tool": "bash", "hint": "Run command?"}}

// 命令
{"cmd": "name", "name": "MyDevice"}
{"cmd": "owner", "name": "Alice"}
{"cmd": "status"} → {"status": {...}}
{"cmd": "unpair"}

// 权限响应
{"id": "abc", "decision": "once|always|deny"}
```

### 状态机

- **连接状态**: OFFLINE → CONNECTED → PAIRING → CONFIRMING
- **角色状态**: SLEEP, IDLE, BUSY, ATTENTION, DIZZY, HEART, CELEBRATE, PAIRING, CONFIRMATION
- **页面**: HOME, PET, INFO, TRANSCRIPT, SETTINGS

---

*报告生成: REA (Reverse Engineering Anything) v6.7.0*  
*分析人员: Hermes Agent + Simple Yu*
