# tec-SDMEM
TEC-1 SD Memory Interface


- https://uditagarwal.in/understanding-sd-sdio-and-mmc-interface/
- https://forums.parallax.com/discussion/99751/c64-datasette-emulator-with-the-propeller
- https://www.zx81keyboardadventure.com/2017/08/model-100-micro-sd-card-reader-mytesed.html
- 




Excellent question — and this one goes right to the heart of interfacing *modern* storage with *vintage* 8-bit systems like your TEC-1 (Z80 SBC).
Let’s break it down into **three layers**: electrical interface, protocol level, and firmware/software level.

---

## 🧩 1. Electrical / I/O Level

### ⚙️ Physical interface

An SD card can operate in two modes:

| Mode                 | Pins used                        | Description                                                               |
| -------------------- | -------------------------------- | ------------------------------------------------------------------------- |
| **SPI mode**         | CS, MOSI, MISO, SCK, +3.3 V, GND | Simple serial bus (works perfectly with Z80 I/O ports).                   |
| **SD (native) mode** | CMD, CLK, DAT0–3, +3.3 V, GND    | High-speed parallel, requires full SD host logic — too complex for TEC-1. |

So you always use **SPI mode** with 8-bit micros.

### ⚡ Level shifting

SD cards are **3.3 V logic**, but the TEC-1 I/O ports are **5 V TTL**.
You’ll need:

* 74LVC125 or 4050 for **Z80→SD (MOSI, SCK, CS)**,
* Direct connection or resistor divider for **SD→Z80 (MISO)**,
* Pull-ups (10 kΩ) on MISO, MOSI, CS.

You can use any of these:

* A dedicated SPI interface (like your LS7366R circuit),
* Bit-banged SPI on a PIO or output latch (74LS595 + 74LS165).

---

## 💾 2. Protocol Level (How the I/O Works)

In **SPI mode**, the SD card acts as an SPI slave.
You send *commands* (CMD0, CMD8, CMD17, etc.) and get *responses*.

### SPI Transaction Example

| Step | Description                   | Command (hex)       | Notes               |
| ---- | ----------------------------- | ------------------- | ------------------- |
| 1    | Power-on (CS high, 80 clocks) | —                   | initialize card     |
| 2    | **CMD0** (GO_IDLE_STATE)      | `40 00 00 00 00 95` | response `01`       |
| 3    | **CMD8** (SEND_IF_COND)       | `48 00 00 01 AA 87` | response `01`       |
| 4    | Loop **ACMD41** until ready   | —                   | response `00`       |
| 5    | **CMD58** (read OCR)          | `7A 00 00 00 00 FD` | get card type       |
| 6    | **CMD17** (read block)        | `51 00 00 00 00 FF` | then 512 bytes data |

---

### Example Bit-Bang I/O Flow (simplified)

```forth
\ Bit-banged SPI write and read (MINT-style pseudocode)
:SCK 1 ;  \ clock pin
:MOSI 2 ; \ data out
:MISO 3 ; \ data in
:CS   4 ; \ chip select

:spi_clk  ( toggle clock )
  SCK 0 /O
  SCK 1 /O
;

:spi_tx ( byte -- )
  8 0 DO
    DUP 0x80 AND IF MOSI 1 /O ELSE MOSI 0 /O THEN
    spi_clk
    SHL
  LOOP DROP
;

:spi_rx ( -- byte )
  0
  8 0 DO
    SHL
    spi_clk
    MISO /I IF 1 OR THEN
  LOOP
;
```

Then you use `spi_tx` and `spi_rx` to form command packets and read responses.

---

## 🧠 3. Firmware / Software Layer

Once initialized, you can:

* Read/write 512-byte **blocks** (like sectors on a floppy).
* Use a **tiny FAT16/FAT32 library** if you want files, or treat it as **raw sectors**.

For the TEC-1, the simplest is:

* Mount SD as raw storage, each 512-byte block mapped by sector number.
* Implement a monitor command like:

```
SDR <sector>    ; read 512 bytes to RAM
SDW <sector>    ; write 512 bytes from RAM
```

---

## 🧱 4. Typical Hardware Connection Example

| SD pin | Connect to         | Notes           |
| ------ | ------------------ | --------------- |
| CS     | PIO bit or /O port | active low      |
| CLK    | PIO bit or /O port | toggled by code |
| MOSI   | PIO bit            | data out        |
| MISO   | PIO bit            | data in         |
| VCC    | 3.3 V regulator    | LM1117-3.3      |
| GND    | common             | —               |

If you already have your SPI routines for the **LS7366R**, you can reuse the same `/O` and `/I` port system — just a different chip select.

---

## 🧩 5. Minimal Z80 Assembly Outline

```asm
; Write byte to SPI
SPI_OUT:
    LD A,(HL)
    OUT (SPI_PORT),A
    JP SPI_WAIT
; Read byte
SPI_IN:
    IN A,(SPI_PORT)
    RET
```

You’ll expand these into routines for `CMD0`, `CMD17`, etc.

---

## 🧰 Summary

| Layer                  | Description         | What you do                          |
| ---------------------- | ------------------- | ------------------------------------ |
| Electrical             | SPI (3.3 V)         | Level shift, wire to Z80 ports       |
| Protocol               | CMD0, CMD17, etc.   | Send via SPI routines                |
| Software               | Sector/block access | Build SD read/write monitor commands |
| File System (optional) | FAT16/FAT32         | Parse directory and file tables      |

---

If you’d like, I can:

* Draw a **pin-to-pin TEC-1 ↔ SD card schematic**,
* Write a **complete Z80/MINT hybrid driver**,
* Or add a **FAT16 directory lister** for use with your existing `/I` and `/O` commands.

👉 Which of those would you like next — the **hardware wiring diagram**, or the **Z80/MINT driver code**?
