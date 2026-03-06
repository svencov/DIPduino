# DIPduino

An inexpensive, through-hole Arduino clone you build from scratch.

DIPduino uses large, easy-to-solder through-hole (DIP) components — perfect for beginners learning to solder. All you need is a soldering iron, solder, and the kit.

- [Assembly instructions (flip-book)](https://heyzine.com/flip-book/8c8d4a5704.html)
- [Kickstarter campaign](https://www.kickstarter.com/projects/dipduino/dipduino-an-arduino-you-build)

## Repository Structure

This repo contains two submodules:

| Submodule | Description |
|-----------|-------------|
| [`DIPduino_KiCad`](https://github.com/svencov/DIPduino_KiCad) | KiCad schematic and PCB design, plus ready-to-order Gerber files |
| [`DIPduino_USB_firmware`](https://github.com/svencov/DIPduino_USB_firmware) | PIC16F1454 USB-to-serial firmware (replaces FTDI/ATmega16U2) |

## Ordering PCBs

The PCB design files are in the `DIPduino_KiCad` submodule.

1. Go to [DIPduino_KiCad](https://github.com/svencov/DIPduino_KiCad)
2. Download the latest Gerber zip — `Gerber_V4.1.zip` is the most recent
3. Upload the zip to a PCB manufacturer:
   - [JLCPCB](https://jlcpcb.com/) — upload the Gerber zip, select quantity, color, and order
   - [PCBWay](https://www.pcbway.com/) — same process
   - [OSH Park](https://oshpark.com/) — drag and drop the zip
4. Default settings (2-layer, 1.6mm, FR4) work fine — no special options needed

If you want to modify the PCB, open `DIPduino.kicad_pcb` and `DIPduino.kicad_sch` in [KiCad](https://www.kicad.org/) (free, open source).

## Flashing the ATmega328P (Main Microcontroller)

The ATmega328P needs the **Arduino Nano (old bootloader)** flashed so it can be programmed from the Arduino IDE over USB. This is the larger ATmegaBOOT bootloader (not Optiboot), which is why the high fuse is `0xDA` (1024-word boot section).

### What You Need

- A **USBasp** programmer (recommended) — or another Arduino running the "Arduino as ISP" sketch
- 6 jumper wires (for the ISP header)
- `avrdude` installed ([download](https://github.com/avrdudes/avrdude/releases) or install via your package manager)

### Fuse Settings

A blank ATmega328P ships with its internal 1 MHz oscillator selected. The fuses **must** be set correctly before the chip can use the external 16 MHz crystal and accept the bootloader.

The correct fuse values for the DIPduino (ATmega328P @ 16 MHz):

| Fuse | Value | Purpose |
|------|-------|---------|
| Low | `0xFF` | External crystal, full-swing oscillator, no clock divide |
| High | `0xDA` | 1024-word boot section, boot reset vector enabled, SPI programming enabled |
| Extended | `0xFD` | Brown-out detection at 2.7 V |

### Steps (Using USBasp + avrdude)

1. **Connect the USBasp** to the DIPduino's ISP header (or wire the 6 ISP pins directly to the ATmega328P)

2. **Set the fuses** — this must be done first so the chip switches to the external 16 MHz crystal:

   ```bash
   avrdude -c usbasp -p m328p \
     -U lfuse:w:0xFF:m -U hfuse:w:0xDA:m -U efuse:w:0xFD:m
   ```

3. **Flash the bootloader** — download [`ATmegaBOOT_168_atmega328.hex`](https://github.com/arduino/ArduinoCore-avr/raw/master/bootloaders/atmega/ATmegaBOOT_168_atmega328.hex) from the official Arduino core, then:

   ```bash
   avrdude -c usbasp -p m328p \
     -U flash:w:ATmegaBOOT_168_atmega328.hex:i
   ```

4. Done — the ATmega328P is now ready.

### Alternative: Using Arduino as ISP

If you don't have a USBasp, you can use another Arduino as the programmer:

1. **Prepare the programmer Arduino**: Open the Arduino IDE → File → Examples → 11.ArduinoISP → ArduinoISP. Upload this sketch to a working Arduino (Uno, Nano, etc.)

2. **Wire the programmer to the DIPduino's ATmega328P**:
   | Programmer Arduino | DIPduino ATmega328P |
   |---|---|
   | Pin 13 (SCK) | SCK (pin 19) |
   | Pin 12 (MISO) | MISO (pin 18) |
   | Pin 11 (MOSI) | MOSI (pin 17) |
   | Pin 10 (SS) | RESET (pin 1) |
   | 5V | VCC |
   | GND | GND |

3. **Set fuses and burn the bootloader**: In the Arduino IDE:
   - Tools → Board → "Arduino Nano"
   - Tools → Processor → "ATmega328P (Old Bootloader)"
   - Tools → Programmer → "Arduino as ISP"
   - Tools → **Burn Bootloader** (this sets the fuses and writes the bootloader in one step)

4. Wait for "Done burning bootloader." — the ATmega328P is now ready.

## Flashing the PIC16F1454 (USB-to-Serial Chip)

The DIPduino uses a **PIC16F1454** instead of an FTDI or ATmega16U2 for USB-to-serial. It's cheaper and available in a through-hole DIP package. The firmware is in the [`DIPduino_USB_firmware`](https://github.com/svencov/DIPduino_USB_firmware) submodule, forked from [Jens Geisler's PIC16F1454_USB2Serial](https://github.com/jgeisler0303/PIC16F1454_USB2Serial).

### Pre-compiled Firmware

A pre-compiled `.hex` file is included: `PIC16F1454_USB2Serial.hex`

### Flashing with an Arduino (ardpicprog)

If you don't have a PIC programmer, you can use an Arduino:

1. Clone [ardpicprog](https://github.com/jgeisler0303/ardpicprog)
2. Follow the [ardpicprog documentation](http://rweather.github.io/ardpicprog/) and the [PIC16F1454 instructions](https://github.com/jgeisler0303/ardpicprog#support-for-16f14545559-added-by-jgeisler0303)
3. Upload the `PIC16F1454_USB2Serial.hex` file to the PIC

### Flashing with a PICkit or MPLAB IPE

1. Install [MPLAB IPE](https://www.microchip.com/en-us/tools-resources/production/mplab-integrated-programming-environment) (free from Microchip)
2. Connect your PICkit programmer to the PIC16F1454
3. Load `PIC16F1454_USB2Serial.hex` and program

### Building the Firmware from Source

If you want to modify the USB firmware:

- **Linux**: Install the [XC8 compiler](https://www.microchip.com/en-us/tools-resources/develop/mplab-xc-compilers) and run `make`
- **Windows/macOS**: Open the project in [MPLAB X IDE](https://www.microchip.com/en-us/tools-resources/develop/mplab-x-ide) (the `MPLAB.X` directory is the project)

## After Flashing Both Chips

1. Assemble the DIPduino following the [instructions](https://heyzine.com/flip-book/8c8d4a5704.html)
2. Connect the DIPduino to your computer via USB
3. Open the Arduino IDE:
   - Tools → Board → "Arduino Nano"
   - Tools → Processor → "ATmega328P (Old Bootloader)"
4. Select the correct port
5. Upload a sketch — it works just like a regular Arduino Nano

## Credits

- Designed by Stephen Covrig
- USB firmware by [Jens Geisler](https://github.com/jgeisler0303) (PIC16F1454_USB2Serial)
- Inspired by Nkosana Masuku's vision for affordable STEM education in Zimbabwe
- Backed by 36 supporters on [Kickstarter](https://www.kickstarter.com/projects/dipduino/dipduino-an-arduino-you-build)
- All glory to Jesus Christ
