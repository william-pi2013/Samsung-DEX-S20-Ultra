# Samsung DeX S20 Ultra + Raspberry Pi Pico OLED Status Display

This project uses a Raspberry Pi Pico and a small I²C OLED to display status text while using Samsung DeX on a Galaxy S20 Ultra. The Pico reads newline-delimited text over USB serial and renders up to three lines on the display.

> **Important:** The Pico display and its MicroPython code are intended as a simple serial-text demo. The sample Android script reports RAM usage and the current time; CPU is a placeholder, not a measured utilization value. Android USB serial access also varies by device and Android version, so the phone-to-Pico connection may need additional setup.

## What you’ll need

### DeX setup

- Samsung Galaxy S20 Ultra
- HDMI monitor or TV
- USB-C hub/dock with HDMI, at least two USB-A ports, and USB-C Power Delivery (PD) input
- USB keyboard and mouse
- USB-C charger rated at least 25 W (recommended)
- HDMI cable

### OLED display setup

- Raspberry Pi Pico or Pico W
- I²C SSD1306 OLED (128×64 as used in the example; 128×32 also works with a code adjustment)
- Breadboard and four jumper wires
- Micro-USB data cable for the Pico

## 1. Connect the phone to DeX

1. Connect the charger to the hub’s USB-C PD input.
2. Connect the keyboard and mouse to the hub’s USB-A ports.
3. Connect the hub to the monitor with an HDMI cable.
4. Connect the hub’s USB-C cable to the Galaxy S20 Ultra.
5. If the monitor mirrors the phone instead of showing DeX, open the notification shade, tap the DeX notification or quick setting, and select the external display option.

Exact menu names and behavior can vary with the phone’s software version and hub.

## 2. Wire the OLED to the Pico

Use the Pico’s **physical pin numbers** below. Check your board’s pinout before connecting power.

| OLED | Pico physical pin | Pico signal | Purpose |
|---|---:|---|---|
| GND | 38 | GND | Ground |
| VCC | 36 | 3V3(OUT) | 3.3 V power |
| SCL | 2 | GP1 / I²C0 SCL | I²C clock |
| SDA | 1 | GP0 / I²C0 SDA | I²C data |

Power the display from 3.3 V unless its manufacturer specifies otherwise. Do not connect the OLED to 5 V unless the display is explicitly rated for it.

## 3. Install MicroPython and the OLED driver

1. Hold **BOOTSEL** while connecting the Pico to a computer over USB. Release BOOTSEL when the `RPI-RP2` drive appears.
2. Download the MicroPython UF2 for your Pico from the [official Raspberry Pi documentation](https://www.raspberrypi.com/documentation/microcontrollers/micropython.html) and copy it to `RPI-RP2`. The Pico will reboot.
3. Open Thonny (or another MicroPython-compatible IDE) and connect to the Pico.
4. Install an SSD1306 MicroPython driver, or save `ssd1306.py` onto the Pico.
5. Save the following as `main.py` on the Pico.

```python
import machine
import select
import sys
import ssd1306

# GP0 = SDA, GP1 = SCL

i2c = machine.I2C(
    0,
    sda=machine.Pin(0),
    scl=machine.Pin(1),
    freq=400000,
)
oled = ssd1306.SSD1306_I2C(128, 64, i2c)


def update_display(line1, line2, line3):
    oled.fill(0)
    oled.text("--- DEX STATS ---", 0, 0)
    oled.text(line1[:16], 0, 20)
    oled.text(line2[:16], 0, 35)
    oled.text(line3[:16], 0, 50)
    oled.show()


update_display("Waiting for", "DeX data...", "")

# Expected input: one comma-separated, newline-terminated message, e.g.
# CPU: N/A,RAM: 62%,Time: 12:00:00
poll = select.poll()
poll.register(sys.stdin, select.POLLIN)
buffer = ""

while True:
    if poll.poll(100):
        char = sys.stdin.read(1)
        if not char:
            continue
        if char == "\n":
            fields = buffer.strip().split(",")
            if len(fields) >= 3:
                update_display(fields[0], fields[1], fields[2])
            buffer = ""
        else:
            buffer += char
            # Avoid unbounded growth if the sender never terminates a line.
            if len(buffer) > 128:
                buffer = ""
```

This example assumes the Pico receives the sender’s bytes on its USB serial input. MicroPython’s USB serial/REPL behavior can differ by firmware and host; test the serial path before relying on it. If the REPL consumes input or the Pico does not appear as a serial device, use a supported USB-serial approach for your setup or connect the phone and Pico over UART with the required level-safe wiring.

## 4. Send sample status text from Android

Install Termux from a trusted source such as [F-Droid](https://f-droid.org/packages/com.termux/) or the [official Termux GitHub project](https://github.com/termux/termux-app). In Termux, install Python and PySerial:

```sh
pkg update
pkg install python
python -m pip install pyserial
```

The following example sends available-memory percentage and the current time. CPU is deliberately shown as `N/A`; it does not calculate CPU utilization.

```python
import re
import time
import serial

# Change this to the serial port that is actually available on your setup.
PORT = "/dev/ttyACM0"
BAUD = 115200


def get_mem_usage():
    try:
        with open("/proc/meminfo", "r", encoding="utf-8") as file:
            meminfo = file.read()
        total_match = re.search(r"MemTotal:\s+(\d+)", meminfo)
        available_match = re.search(r"MemAvailable:\s+(\d+)", meminfo)
        if not total_match or not available_match:
            return "RAM: N/A"
        total = int(total_match.group(1))
        available = int(available_match.group(1))
        if total <= 0:
            return "RAM: N/A"
        used_percent = round((total - available) / total * 100)
        return f"RAM: {used_percent}%"
    except (OSError, ValueError):
        return "RAM: N/A"


try:
    with serial.Serial(PORT, BAUD, timeout=1) as ser:
        while True:
            payload = f"CPU: N/A,{get_mem_usage()},{time.strftime('Time: %H:%M:%S')}\n"
            ser.write(payload.encode("utf-8"))
            time.sleep(2)
except serial.SerialException as error:
    print(f"Could not open {PORT}: {error}")
    print("Check USB host permissions, the device path, and serial access on your Android setup.")
```

Save the script as `send_stats.py`, connect the Pico to the hub with a **data-capable** USB cable, then run:

```sh
python send_stats.py
```

### Android USB notes

A USB device connected to an Android phone does not necessarily appear as `/dev/ttyACM0`. Android may require you to grant USB host permission, and standard Termux/PySerial may not have access to the device as a Linux serial port. If no usable port is available, this script cannot communicate with the Pico as written; use an Android USB host/serial application or a supported serial bridge and adapt the sender accordingly. Do not assume the port name—check what your Android setup exposes.

## Troubleshooting

- **OLED stays blank:** Check GND/VCC and SDA/SCL wiring, confirm the display is SSD1306-compatible, and verify the I²C address (commonly `0x3C`).
- **Display is clipped:** The example is set to 128×64 and shows three short lines. For a 128×32 OLED, change the driver dimensions and adjust the text positions.
- **Pico does not appear on the phone:** Confirm the cable supports data, the hub supports USB host connections, and Android has granted any requested USB permission.
- **Serial port cannot be opened:** Check Android’s USB serial support and the actual device path. `/dev/ttyACM0` is only an example.
- **No updates reach the OLED:** Confirm the Pico program is running and that the sender transmits comma-separated fields ending in `\n`.
- **CPU displays `N/A`:** This is expected in the sample. Implement a suitable Android-specific CPU metric source before changing it to a measured value.
