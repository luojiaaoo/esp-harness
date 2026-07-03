---
name: esp-serial-port
description: Operate ESP8266 or ESP32 boards over a Windows USB serial port for MicroPython development. Use when the task involves preparing the serial environment, scanning COM ports, checking chip or firmware configuration, stopping a running MicroPython program with Ctrl-C over serial, erasing flash, flashing firmware with esptool, or quickly executing MicroPython REPL expressions such as repr() through serial.
---

# ESP Serial Port

Use this skill for ESP8266/ESP32 serial work from Windows PowerShell. Keep operations scoped to serial discovery, firmware operations, and quick MicroPython REPL execution.

## 环境准备

Confirm Python and serial tooling:

```powershell
python --version
python -m esptool version
```

For MicroPython REPL access from host scripts, ensure `pyserial` exists:

```powershell
python -c "import serial; print('pyserial OK')"
```

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
python -m esptool --port COM3 chip_id
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
python -m esptool --port COM3 erase_flash
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
python -m esptool --port COM3 --baud 460800 write_flash --flash_size=detect 0 C:\path\to\firmware.bin
```

If high-speed flashing is unstable, use:

```powershell
python -m esptool --port COM3 --baud 115200 write_flash --flash_size=detect 0 C:\path\to\firmware.bin
```

Success should include:

```text
Wrote ... bytes
Verifying written data...
Hash of data verified
```

## 快速 repr 执行

Use a short host-side Python script to stop any running `main.py`, execute one REPL expression, and print the result. Set `DTR` and `RTS` false to avoid unwanted reset/boot toggles.

```powershell
@'
import serial, time, sys

port = 'COM3'
expr = "repr('abc')"

with serial.Serial(port, baudrate=115200, timeout=0.3) as ser:
    ser.dtr = False
    ser.rts = False
    time.sleep(0.5)
    ser.write(b'\x03\x03')
    time.sleep(0.5)
    ser.reset_input_buffer()
    ser.write(("print(" + expr + ")\r\n").encode("utf-8"))
    time.sleep(1)
    sys.stdout.buffer.write(ser.read(4096))
'@ | python -
```

Expected output:

```text
'abc'
>>>
```

For arbitrary commands, send a single-line MicroPython command ending in `\r\n`. If the REPL enters `...` continuation mode, send `Ctrl-C` twice and retry with a single-line command.
