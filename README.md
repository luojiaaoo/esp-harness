# eps-harness

适用于 ESP8266 / ESP32 嵌入式板的 MicroPython 开发工具集，提供串口操作、固件刷写、REPL 调试等一站式能力。

## 技能列表

| 技能 | 说明 |
|------|------|
| [esp-serial-port](./skills/esp-serial-port/SKILL.md) | ESP8266/ESP32 串口操作：端口扫描、芯片检测、固件刷写与擦除、REPL 快速执行 |

## 功能概览

- **端口扫描** — 自动发现 COM 端口，识别 CP210x / CH340 等 USB-UART 设备
- **芯片检测** — 通过 `esptool` 读取芯片 ID 及 MicroPython 版本信息
- **固件刷写** — 支持擦除 Flash 并写入 MicroPython 固件
- **REPL 交互** — 通过串口发送 Ctrl-C 中止运行中的程序，执行单行 MicroPython 表达式

## 环境要求

- Windows 系统（PowerShell）
- Python 3.x
- `esptool` — `pip install esptool`
- `pyserial` — `pip install pyserial`
- 对应 USB-UART 驱动（CP210x / CH340）

## 快速开始

```powershell
# 1. 检查环境
python --version
python -m esptool version
python -c "import serial; print('pyserial OK')"

# 2. 扫描端口
[System.IO.Ports.SerialPort]::GetPortNames()

# 3. 检测芯片
python -m esptool --port COM3 chip_id

# 4. 擦除 Flash 并刷写固件
python -m esptool --port COM3 erase_flash
python -m esptool --port COM3 --baud 460800 write_flash --flash_size=detect 0 firmware.bin
```

## 项目结构

```
eps-harness/
├── skills/
│   └── esp-serial-port/
│       └── SKILL.md          # 串口操作技能定义
├── .vscode/
│   └── settings.json
├── .gitignore
├── LICENSE
└── README.md
```

## 许可

本项目基于 MIT 许可协议开源，详见 [LICENSE](./LICENSE)。
