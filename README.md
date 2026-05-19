# Cyberdeck_RPI2b
DIY cyberdeck build with a RPi 2B, a generic 3,5inch GPIO screen, two wifi adapter (T3U) and GPS module

# Raspberry Pi 2B Full Setup Overview

Configuration summary for:

- Raspberry Pi 2B
- Raspberry Pi OS Bullseye 32-bit
- 3.5" SPI TFT display (480x320)
- XPT2046/ADS7846 touchscreen
- CLI-first system
- Bluetooth adapter
- Bluetooth keyboard
- Dual Wi-Fi adapters
- USB GPS receiver

---

# 1. Basic OS installation

Installation of the SD-card
- Get the OS image with drivers for the screen from:
https://www.waveshare.com/wiki/3.5inch_RPi_LCD_(A)
- Since the RPi2B is used: https://drive.google.com/file/d/1imrfzcxzSjnNLcWMAivFNQF_kZm0zPJT/view?usp=sharing
- User Raspberry Pi Imager to flash the OS on the SD card. 

Notes:
- The Waveshare image already contains working TFT support.
- Installing the `LCD-show` drivers later modifies framebuffer and boot configuration again.
- A standard Raspberry Pi OS Bullseye image may also work, but this was not tested.

# 2. Base Raspberry Pi OS Setup

Before updating the system, make sure the Raspberry Pi is accesable over SSH!
When SSH login is possible, update system:

```bash
sudo apt update
sudo apt upgrade -y
```

Enable SPI + UART:

```bash
sudo raspi-config
```

Enable:
- SPI
- Serial port hardware

Disable:
- serial login shell

---

Reboot:

```bash
sudo reboot
```
# 3. TFT Display Setup (3.5" SPI LCD)
After the update, the display stops working. To fix the display again, drivers are installed from https://github.com/goodtft/LCD-show.




```bash
sudo rm -rf LCD-show
git clone https://github.com/goodtft/LCD-show.git
chmod -R 755 LCD-show
cd LCD-show/
# Depending on the used screen, the next command can be different
sudo ./LCD35-show
```

## 3.1 Replace /boot/config.txt

The display uses:
- SPI framebuffer
- fbtft driver
- ILI9486 controller
- XPT2046/ADS7846 touch controller

## `/boot/config.txt`

Working configuration:

```ini
# Enable SPI and UART
dtparam=spi=on
enable_uart=1

# Disable overscan for correct TFT alignment
disable_overscan=1

# Allow two framebuffers: HDMI + TFT
max_framebuffers=2

# TFT overlay settings
dtoverlay=piscreen,speed=16000000,rotate=270,fps=20

# Optional: HDMI output can be disabled or left as fallback
# hdmi_force_hotplug=1
# hdmi_group=2
# hdmi_mode=87
# hdmi_cvt 480 320 60 6 0 0 0
# hdmi_drive=2

```

Notes:
- `rotate=270` rotates display
- framebuffer becomes `fb1`
- SPI speed = 16 MHz

---

## 3.2 Console Boot Only (No GUI)

prerequisites:
- boot directly to CLI
- no desktop auto-start



## `/boot/cmdline.txt`

Working version:

```text
console=tty1 root=/dev/mmcblk0p2 rootfstype=ext4 elevator=deadline fsck.repair=yes rootwait fbcon=font:SUN12x22 systemd.unit=multi-user.target

```

Important:
- The entire line of code is one line of code
- `fbcon=map:1` Do not use! causes flickering of screen
- `fbcon=font:VGA8x8` is to small. 
---

Reboot:

```bash
sudo reboot
```

After reboot the display should be working again.
# 4. Bluetooth Setup

USB Bluetooth adapter:
- TP-Link RTL8761BU

Kernel driver:
- `btusb`


## Install tools

```bash
sudo apt install bluetooth bluez blueman
```

TUI manager:

```bash
sudo apt install bluetui
```

## Pairing Bluetooth keyboard

Getting the keyboard to work is still a mistery. The used keyboard is a "M7 Mini Bluetooth Keyboard"  from Aliexpres. According to the manual a pairing code should be entered on the keyboard, but this code never showed. Swithing to desktop and using the GUI did not seem to work. The RPi said it was connected to the keyboard but the keyboard still appeared to be in pairing mode. After a reboot 'somehow' worked. Any suggestions on repeatable working method are welcome.

Start:

```bash
bluetoothctl
```

Commands:

```bash
power on
agent on
default-agent
scan on
pair MAC
trust MAC
connect MAC
```
Reboot:

```bash
sudo reboot
```


# 5. Wi‑Fi Adapter Setup

You used two adapters:

| Adapter | Chipset | Driver |
|---|---|---|
| TP-Link Archer T3U | RTL8822BU | rtl88x2bu |
| RTL8188CUS adapter | RTL8188CUS | rtl8192cu |

---

## 5.1 Archer T3U Setup (rtl88x2bu)

Adapter:
- TP-Link Archer T3U
- USB ID: `2357:0138`

Driver:
- DKMS `rtl88x2bu`

## Install dependencies

```bash
sudo apt install -y dkms git raspberrypi-kernel-headers build-essential
```

## Build driver

Example:

```bash
git clone https://github.com/morrownr/88x2bu-20210702.git
cd 88x2bu-20210702
sudo ./install-driver.sh
```

## Result

Driver loaded:

```bash
lsmod | grep 88x2bu
```

Interface:
- `wlan0`

---

## RTL8188CUS Adapter Fix

Problem:
- adapter detected by USB
- no wlan interface appeared

Cause:
- wrong driver (`rtl8xxxu`) auto-loaded

## 5.2 Dual adapter fix
Somehow not both adapter where shown and getting a device name. This termporary fixed is used to check if installation of both adapter drivers worked:

```bash
sudo modprobe -r rtl8xxxu
sudo modprobe rtl8192cu
```

This created:
- `wlan1`

---

## 5.3 Permanent Wi‑Fi Fix
if this works blacklist the bad drive, force the good drive and give both adapters a stable name

## Blacklist bad driver

Create:

```bash
sudo nano /etc/modprobe.d/blacklist-rtl8xxxu.conf
```

Contents:

```text
blacklist rtl8xxxu
```

## Force correct driver

Create:

```bash
sudo nano /etc/modules-load.d/rtl8192cu.conf
```

Contents:

```text
rtl8192cu
```

## Rebuild initramfs

```bash
sudo update-initramfs -u
```

---

## Stable Interface Naming

Optional but recommended.

Create:

```bash
sudo nano /etc/udev/rules.d/70-persistent-net.rules
```

Example:

```text
SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="20:23:51:79:25:88", NAME="wlan0"

SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="00:0b:81:94:fc:db", NAME="wlan1"
```

This keeps:
- Archer T3U = wlan0
- RTL8188CUS = wlan1

---

Reboot:

```bash
sudo reboot
```

and check if the interfaces are showing with 

```bash
ip a
``` 

## 5.4 Wi‑Fi Connection Tools

CLI/TUI tools:

## nmtui

Install:

```bash
sudo apt install network-manager
```

Use:

```bash
nmtui
```

---

another option is to use, but a TUI like Impala is preffered.

```bash
sudo raspi-config
```

# 6. GPS Setup

GPS:
- u-blox 7 USB receiver
- `/dev/ttyACM0`

Detected through:
- `cdc_acm`

## Install gpsd

```bash
sudo apt install gpsd gpsd-clients
```

## Manual test

Start daemon:

```bash
sudo gpsd /dev/ttyACM0 -F /run/gpsd.sock
```

## Test tools

```bash
cgps -s
```

or:

```bash
gpsmon
```

---

## GPS Diagnosis

Raw GPS data confirmed:

```bash
cat /dev/ttyACM0
```

Output showed:
- valid NMEA
- u-blox firmware
- antenna OK

But:
- no satellite fix indoors

Meaning:
- GPS hardware works correctly
- needs better sky visibility

---

## Permanent gpsd Configuration

Edit:

```bash
sudo nano /etc/default/gpsd
```

Set:

```ini
START_DAEMON="true"
GPSD_OPTIONS="-n"
DEVICES="/dev/ttyACM0"
USBAUTO="false"
```

Enable:

```bash
sudo systemctl enable gpsd
sudo systemctl restart gpsd
```

---

Reboot:

```bash
sudo reboot
```
# 7. SDR Setup

The cyberdeck can be use with a RTL-SDRv4 dongle. Basic setup is required:

```bash
sudo apt install rtl-sdr librtlsdr0 rtl-433
```

# 8. Useful Diagnostic Commands

## Wi‑Fi

```bash
iw dev
ip a
rfkill list
lsmod | grep rtl
```

## USB

```bash
lsusb
lsusb -t
usb-devices
```

## GPS

```bash
cgps -s
gpsmon
cat /dev/ttyACM0
```

## Power

```bash
vcgencmd get_throttled
```

Expected good value:

```text
throttled=0x0
```

---

# 9. Final System State

Basic working setup of the cyberdeck is complete 

| Component | Status |
|---|---|
| SPI TFT display | Working |
| Touchscreen | Working |
| Console on TFT | Working |
| CLI boot | Working |
| Bluetooth | Working |
| Bluetooth keyboard | Working |
| Archer T3U Wi‑Fi | Working |
| RTL8188CUS Wi‑Fi | Working |
| Dual Wi‑Fi coexistence | Working |
| GPS serial output | Working |
| gpsd | Working |
| Satellite fix | depends on sky visibility |
# Cyberdeck_RPI2b
DIY cyberdeck build with a RPi 2B, a generic 3,5inch GPIO screen, two wifi adapter (T3U) and GPS module
