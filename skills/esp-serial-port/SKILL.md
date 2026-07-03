---
name: esp-serial-port
description: ESP8266/ESP32 MicroPython development over Windows USB serial: scan COM ports, inspect chip info, flash/erase firmware with esptool, and transfer/run/deploy files with ampy.
---

# ESP Serial Port

Use this skill for ESP8266/ESP32 MicroPython development from Windows PowerShell — serial discovery, esptool firmware operations, ampy file operations, and run-vs-deploy workflow (app.py for dev, main.py for production).

## 环境准备

Confirm Python and serial tooling:

```powershell
python -m esptool version
```

For MicroPython REPL access from host scripts, ensure `pyserial` exists:

```powershell
python -c "import serial; print('pyserial OK')"
```

For file transfer, install `ampy`:

```powershell
pip install adafruit-ampy
```

**IMPORTANT**: Call `ampy` directly as a CLI tool, NOT with `python -m ampy` (the latter will fail).

If the board uses CP210x/CH340 and no COM port appears, install the matching USB-UART driver first.

## 扫描端口

List available COM ports:

```powershell
[System.IO.Ports.SerialPort]::GetPortNames()
```

Inspect likely ESP USB-UART devices:

```powershell
Get-PnpDevice -PresentOnly | Where-Object {
  $_.FriendlyName -match 'CP210|CH340|USB-SERIAL|UART|Serial|COM'
} | Select-Object Status, Class, FriendlyName, InstanceId
```

Typical names:

- `Silicon Labs CP210x USB to UART Bridge (COMx)`
- `USB-SERIAL CH340 (COMx)`
- `FTDI USB Serial Port (COMx)`

If opening the port returns `PermissionError(13)` or `拒绝访问`, another app owns the port. Ask the user to close Thonny, Arduino IDE Serial Monitor, PuTTY, VS Code serial plugins, or other COM-port tools.

## 查看配置

Check chip identity with `esptool`:

```powershell
python -m esptool --port COM3 chip-id
```

For MicroPython firmware/version checks, use REPL:

```python
import sys
print(sys.version)
print(sys.platform)
```

Expected ESP8266 platform output:

```text
esp8266
```

Use `COM3` only if that is the discovered board port; otherwise substitute the actual `COMx`.

List files already on the board:

```powershell
ampy --port COM3 ls
ampy --port COM3 ls -l
```

## 中止当前程序

If a flashed `main.py` or current MicroPython program keeps running and the REPL does not respond, send `Ctrl-C` over serial before doing REPL or file operations.

Minimal stop signal:

```python
ser.write(b'\x03\x03')
```

PowerShell one-shot:

```powershell
@'
import serial, time, sys

port = 'COM3'

with serial.Serial(port, baudrate=115200, timeout=0.3) as ser:
    ser.dtr = False
    ser.rts = False
    time.sleep(0.5)
    ser.write(b'\x03\x03')
    time.sleep(0.5)
    ser.write(b'\r\n')
    time.sleep(0.5)
    sys.stdout.buffer.write(ser.read(4096))
'@ | python -
```

Expected result is a `>>>` prompt. If the REPL is in continuation mode (`...`), send `Ctrl-C` twice again and retry with a single-line command.

## 擦除 Flash

Erase the board flash:

```powershell
python -m esptool --port COM3 erase-flash
```

Success usually includes:

```text
Detecting chip type... ESP8266
Flash memory erased successfully
Hard resetting via RTS pin
```

If connection fails, retry after unplugging/replugging the board or entering bootloader mode with the board's `BOOT/FLASH` button.

## 刷写固件

Flash MicroPython firmware to address `0x0`:

```powershell
python -m esptool --port COM3 --baud 460800 write-flash --flash-size=detect 0 C:\path\to\firmware.bin
```

If high-speed flashing is unstable, use:

```powershell
python -m esptool --port COM3 --baud 115200 write-flash --flash-size=detect 0 C:\path\to\firmware.bin
```

Success should include:

```text
Wrote ... bytes
Verifying written data...
Hash of data verified
```

## 文件操作

Upload, download, run scripts via `ampy`. If `main.py` is running in a loop, stop it first (see [中止当前程序](#中止当前程序)).

```powershell
# Upload to board
ampy --port COM3 put main.py

# Download from board
ampy --port COM3 get boot.py > boot_backup.py

# Run script without writing to Flash (output printed to terminal)
ampy --port COM3 run test.py

# Delete / create directory
ampy --port COM3 rm old.py
ampy --port COM3 mkdir lib

# Batch upload all .py files
Get-ChildItem *.py | ForEach-Object { ampy --port COM3 put $_.Name }
```

If ampy hangs, the port may be locked — close other serial monitors and retry. Use `--baud 115200` if the default baud rate causes issues.

## 执行程序

### 运行（临时执行）

本地入口文件**必须命名为 `app.py`**，禁止使用 `main.py`——避免 `boot.py` 自动引导启动。用户说"运行"时，先将整个项目上传到板子，再执行入口：

```powershell
# 1. 中止当前程序（见 中止当前程序）

# 2. 上传所有项目文件（app.py 保持原名）
Get-ChildItem *.py,*.json,*.html | ForEach-Object { ampy --port COM3 put $_.Name }

# 3. 临时运行入口脚本
ampy --port COM3 run app.py
```

脚本在板子上执行，输出打印到终端。`app.py` 保持原名不会触发自动启动。后续若仅修改了 `app.py`，可以直接再次 `run` 跳过上传步骤。

### 烧写（部署到板子）

用户明确说"烧写"时，将整个项目部署到板子，上电自动运行。与"运行"的唯一区别：`app.py` 重命名为 `main.py` 使其被 `boot.py` 自动引导：

```powershell
# 1. 中止当前程序（见 中止当前程序）

# 2. 上传 app.py → main.py
ampy --port COM3 put app.py main.py

# 3. 上传其余所有文件
Get-ChildItem *.py,*.json,*.html -Exclude app.py | ForEach-Object { ampy --port COM3 put $_.Name }
```

烧写后板子复位即自动执行 `main.py`。
