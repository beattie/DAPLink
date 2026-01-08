# STM32F103 Blue Pill DAPLink Probe

While it is possible to use a single pin on an STM32F103 to implement SWD input and output (SWDIO)
when implementing an SWD probe, it is more reliable to use two pins and connect them through a resistor.
The DAPLink implementation for the STM32F103 uses PB12 as the SWD input and PB14 as the SWD output.
One way to implement this in a DAPLink probe, with an STM32F103 bluepill, is to solder a 100 Ohm resistor
between PB12 and PB14 making PB14 an SWDIO.
