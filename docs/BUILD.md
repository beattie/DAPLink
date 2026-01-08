# Building and Flashing DAPLink Firmware

This guide covers building and flashing DAPLink firmware for various hardware interface circuits (HICs).

## Prerequisites

### Required Tools

- Python 3.x
- Git
- GNU Arm Embedded Toolchain (recommended: 10.3-2021.10)
- OpenOCD or pyOCD (for flashing)
- A debug probe (e.g., Raspberry Pi Pico with Debugprobe firmware)

### Installation

```bash
# Install virtualenv if not already installed
pip install virtualenv

# Clone the repository
git clone https://github.com/ARMmbed/DAPLink
cd DAPLink

# Create and activate virtual environment
python -m venv venv
source venv/bin/activate  # On Linux/macOS
# OR
venv\Scripts\activate.bat  # On Windows

# Install Python dependencies
pip install -r requirements.txt
```

## Building the Firmware

### STM32F103x8 (64KB Flash) Example

This example builds the interface firmware for STM32F103C8 devices.

#### Clean Build

```bash
# Remove previous build artifacts
rm -rf projectfiles/make_gcc_arm/stm32f103x8_if/
```

#### Build

```bash
# Activate virtual environment (if not already active)
source venv/bin/activate

# Build the interface firmware
python tools/progen_compile.py -t make_gcc_arm stm32f103x8_if
```

**Build output location:**
```
projectfiles/make_gcc_arm/stm32f103x8_if/build/
├── stm32f103x8_if.elf          # ELF file with debug symbols
├── stm32f103x8_if.bin          # Raw binary (intermediate)
├── stm32f103x8_if_crc.bin      # Final binary with CRC32 ✅ USE THIS
├── stm32f103x8_if_crc.hex      # Intel HEX format with CRC32
└── stm32f103x8_if_crc.txt      # CRC32 checksum value
```

#### Build Options

```bash
# Build with verbose output
python tools/progen_compile.py -t make_gcc_arm -v stm32f103x8_if

# Clean build (remove existing and rebuild)
python tools/progen_compile.py -t make_gcc_arm --clean stm32f103x8_if

# Build all projects
python tools/progen_compile.py -t make_gcc_arm

# Build specific HIC with different toolchain
python tools/progen_compile.py -t make_armcc stm32f103x8_if
```

### Other HIC Variants

#### STM32F103xB (128KB Flash)

```bash
# Bootloader
python tools/progen_compile.py -t make_gcc_arm stm32f103xb_bl

# Interface firmware
python tools/progen_compile.py -t make_gcc_arm stm32f103xb_if
```

#### Other HICs

```bash
# List all available projects
grep -A1 "^projects:" projects.yaml

# Examples:
python tools/progen_compile.py -t make_gcc_arm k20dx_if
python tools/progen_compile.py -t make_gcc_arm lpc11u35_if
python tools/progen_compile.py -t make_gcc_arm sam3u2c_if
```

## Flashing the Firmware

### Hardware Setup

Connect your debug probe to the target STM32F103C8:

```
Debug Probe (e.g., Pico Debugprobe) → STM32F103C8
────────────────────────────────────────────────
GND                                   → GND
SWCLK (GP2 on Pico)                  → SWCLK
SWDIO (GP3 on Pico)                  → SWDIO
3V3 (optional)                       → 3.3V
```

### Option 1: Using OpenOCD (Recommended)

#### Check Connection

```bash
# Test connection to target
openocd -f interface/cmsis-dap.cfg \
        -c "adapter speed 1000" \
        -f target/stm32f1x.cfg \
        -c "init" \
        -c "reset halt" \
        -c "shutdown"
```

#### Flash Firmware

```bash
# Flash and verify in one command
openocd -f interface/cmsis-dap.cfg \
        -c "adapter speed 1000" \
        -f target/stm32f1x.cfg \
        -c "program projectfiles/make_gcc_arm/stm32f103x8_if/build/stm32f103x8_if_crc.bin verify reset exit 0x08000000"
```

#### Detailed Flashing (Step-by-Step)

```bash
openocd -f interface/cmsis-dap.cfg \
        -c "adapter speed 1000" \
        -f target/stm32f1x.cfg \
        -c "init" \
        -c "reset halt" \
        -c "flash write_image erase projectfiles/make_gcc_arm/stm32f103x8_if/build/stm32f103x8_if_crc.bin 0x08000000" \
        -c "verify_image projectfiles/make_gcc_arm/stm32f103x8_if/build/stm32f103x8_if_crc.bin 0x08000000" \
        -c "reset run" \
        -c "shutdown"
```

### Option 2: Using pyOCD

#### Check Connection

```bash
# List connected probes
pyocd list

# List supported targets
pyocd list --targets | grep -i stm32
```

#### Flash Firmware

```bash
# Flash using pyOCD (if target is supported)
pyocd flash -t stm32f103rc projectfiles/make_gcc_arm/stm32f103x8_if/build/stm32f103x8_if_crc.bin

# Note: Use stm32f103rc as target type since stm32f103c8 is not in pyOCD's built-in list
# The RC variant is compatible and will work for C8 devices
```

### Verify Flashing

After flashing, verify with OpenOCD:

```bash
openocd -f interface/cmsis-dap.cfg \
        -f target/stm32f1x.cfg \
        -c "init" \
        -c "reset halt" \
        -c "verify_image projectfiles/make_gcc_arm/stm32f103x8_if/build/stm32f103x8_if_crc.bin 0x08000000" \
        -c "shutdown"
```

## Testing the Firmware

### USB Enumeration

After flashing, disconnect the debug probe and connect the STM32F103C8 to USB.

```bash
# Check USB enumeration
lsusb | grep -i "arm\|dap\|0d28"

# Expected output:
# Bus XXX Device XXX: ID 0d28:0204 NXP ARM mbed
```

### Detailed Device Info

```bash
# Get detailed USB device information
lsusb -d 0d28:0204 -v | grep -E "idVendor|idProduct|iManufacturer|iProduct|iSerial|bInterfaceClass"
```

### Expected Interfaces

The STM32F103x8 DAPLink firmware should enumerate with:

1. **CMSIS-DAPv1 HID** - Human Interface Device (debug interface)
2. **CMSIS-DAPv2 WinUSB** - Vendor-specific bulk interface (faster debug)
3. **CDC-ACM** - Virtual serial port
4. **WebUSB** - Web-based debugging interface

**Note:** STM32F103x8 variant does NOT include MSC (Mass Storage) to fit in 64KB flash.

### Serial Port

```bash
# Check for virtual serial port
ls /dev/ttyACM*

# Connect to serial port (example)
screen /dev/ttyACM0 115200
```

## Troubleshooting

### Build Errors

#### Missing Python Dependencies

```bash
# Reinstall requirements
pip install -r requirements.txt --force-reinstall
```

#### CMSIS Header Errors

```bash
# Error: "__CORTEX_M not defined!!"
# Solution: Ensure device.h includes the correct INTERFACE_* macro
# Check: source/hic_hal/device.h
```

#### Binary Too Large

```bash
# Error: "region m_text overflowed"
# Solution: Reduce features or enable LTO in projects.yaml
```

### Flash Errors

#### Target Not Found

```bash
# Error: "Could not find target"
# Solutions:
# 1. Check wiring (GND, SWCLK, SWDIO)
# 2. Verify target has power
# 3. Try lower SWD speed: adapter speed 100
# 4. Check if target is locked or protected
```

#### Connection Failed

```bash
# Try resetting target under reset
openocd -f interface/cmsis-dap.cfg \
        -f target/stm32f1x.cfg \
        -c "init" \
        -c "reset_config srst_only" \
        -c "reset halt" \
        -c "shutdown"
```

#### Verification Failed

```bash
# Flash may be protected - unlock it
openocd -f interface/cmsis-dap.cfg \
        -f target/stm32f1x.cfg \
        -c "init" \
        -c "reset halt" \
        -c "stm32f1x unlock 0" \
        -c "reset halt" \
        -c "shutdown"

# Then retry flashing
```

### USB Enumeration Issues

#### Device Not Detected

1. Check USB cable (must support data, not just power)
2. Verify firmware was flashed correctly
3. Check USB D+/D- connections on hardware
4. Try different USB port
5. Check `dmesg` for USB errors:
   ```bash
   dmesg | tail -50 | grep -i usb
   ```

#### Wrong USB VID/PID

The device should enumerate as:
- VID: `0x0d28` (NXP/Arm Mbed)
- PID: `0x0204` (DAPLink CMSIS-DAP)

If different, check the HIC ID and board configuration.

## Memory Layout Reference

### STM32F103x8 (64KB)

```
Flash (0x08000000 - 0x08010000): 64KB
├─ Interface  (0x08000000 - 0x0800FC00): ~63KB
└─ Config     (0x0800FC00 - 0x08010000): 1KB

RAM (0x20000000 - 0x20005000): 20KB
├─ App RAM    (0x20000000 - 0x20004F00): ~19.75KB
└─ Shared RAM (0x20004F00 - 0x20005000): 256B
```

### STM32F103xB (128KB)

```
Flash (0x08000000 - 0x08020000): 128KB
├─ Bootloader (0x08000000 - 0x0800BC00): 48KB
├─ Interface  (0x0800C000 - 0x0801FC00): 79KB
└─ Config     (0x0801FC00 - 0x08020000): 1KB

RAM (0x20000000 - 0x20005000): 20KB
├─ App RAM    (0x20000000 - 0x20004F00): ~19.75KB
└─ Shared RAM (0x20004F00 - 0x20005000): 256B
```

## Quick Reference Commands

```bash
# Complete build and flash workflow for STM32F103x8
cd /path/to/DAPLink
source venv/bin/activate
python tools/progen_compile.py -t make_gcc_arm --clean stm32f103x8_if
openocd -f interface/cmsis-dap.cfg \
        -c "adapter speed 1000" \
        -f target/stm32f1x.cfg \
        -c "program projectfiles/make_gcc_arm/stm32f103x8_if/build/stm32f103x8_if_crc.bin verify reset exit 0x08000000"
```

## Additional Resources

- [DAPLink Documentation](https://github.com/ARMmbed/DAPLink)
- [STM32F103x8 HIC Details](hic/stm32f103x8.md)
- [STM32F103xB HIC Details](hic/stm32f103xb.md)
- [OpenOCD Documentation](http://openocd.org/doc/)
- [pyOCD Documentation](https://github.com/pyocd/pyOCD)
