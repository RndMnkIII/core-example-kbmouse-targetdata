# Example: Keyboard Mouse Target Data

This example demonstrates using a keyboard and mouse plugged into Dock, as well as reading and writing with APF Target commands. 
By default, the core boots into a gray screen with a mouse cursor. The framebuffer can be drawn on either by using the mouse or keyboard.
In the bottom right of the screen are spaces for 4 small colored boxes indicating the connection status of the 4 controller slots.

On mouse, the controls are 
* LMB: Draw

On keyboard, the controls are
* Arrow keys: Move cursor
* Left CTRL: Draw

On controller 1:
* A/B/X/Y buttons: Reload the framebuffer with 1 out of 4 images read using APF Target commands.
* Select button: Save the framebuffer to slot ID 0x22. The asset file "saved.bin" will be created or overwritten.
* Start button: Load the framebuffer from slot ID 0x22, corresponding to asset file "saved.bin".

Features demonstrated:

* SDRAM controller
* Video generation
* Dock mouse/keyboard
* Cursor generation
* Target read/write commands


## Technical Analysis: Framebuffer storage in saved.bin (data slot 0x22)

### Data Slots — data.json

The core defines two data slots:

| Field          | Slot 0x20 — Image Bank      | Slot 0x22 — Saved Image |
|----------------|-----------------------------|-------------------------|
| `id`           | `0x20`                      | `0x22`                  |
| `filename`     | `ex_image_all.bin`          | `saved.bin`             |
| `required`     | `true`                      | `false`                 |
| `deferload`    | `true`                      | `true`                  |
| `parameters`   | `2` (`0b00000010`)          | `3` (`0b00000011`)      |
| `address`      | `0x00000000`                | —                       |
| `size_maximum` | —                           | `184320`                |

### Parameters Bitmap Decoded

`parameters` is a **bitmap** (not a count). Each bit independently controls one APF behaviour:

| Bit | Meaning when SET           | Slot 0x20 | Slot 0x22 |
|-----|----------------------------|-----------|-----------|
| 0   | User-reloadable            | 0 — no    | **1 — yes** |
| 1   | Core-specific file         | **1 — yes** | **1 — yes** |
| 2–9 | (other flags)              | 0         | 0         |

**Bit 0 — User-reloadable (slot 0x22 only)**  
`saved.bin` appears in Pocket's Core UI Interact menu at runtime, allowing the user to
browse and select a different file. Because `deferload: true` is also set, APF handles
this reload differently from a normal asset: instead of sending host commands `0x0082`
(Data Slot Request Write) + `0x008F` (Data Slot Access All Complete), APF sends only
`0x008A` (Data Slot Update) and refreshes the Dataslot ID/Size table with the new
file's ID and size. The core then reads the new file with a Target command when ready.

**Bit 1 — Core-specific file (both slots)**  
Both asset files are distributed with and stored inside this core's own folder, not
shared across other cores for the same platform.

### deferload and the Dataslot ID/Size Table

Both slots have `deferload: true`, so APF does **not** load their files automatically
on boot. APF still writes each slot's ID and on-disk file size into the Dataslot ID/Size
table located at bridge address `0x2000`–`0x20FF`. This is a 32-entry table where each
entry occupies two 32-bit words:

```
Word 0 [15:0]  — slot ID  (e.g. 0x0022)
Word 1 [31:0]  — file size in bytes (0 if no file is present)
```

The core reads this table to learn the file size of a loaded slot without issuing any
extra Target command.

### Framebuffer Memory Layout

| Property       | Value                                                     |
|----------------|-----------------------------------------------------------|
| Resolution     | 320 × 288 pixels (`VID_H_ACTIVE` × `VID_V_ACTIVE`)        |
| Pixel format   | **RGB565**, 16 bits per pixel                             |
| R field        | bits `[15:11]` — 5 bits                                   |
| G field        | bits `[10:5]`  — 6 bits                                   |
| B field        | bits `[4:0]`   — 5 bits                                   |
| Packing        | 2 pixels per 32-bit SDRAM word                            |
| Word address   | `(row × 160) + (col / 2)`                                 |
| Even column    | lower halfword `[15:0]` of the word                       |
| Odd column     | upper halfword `[31:16]` of the word                      |
| Total size     | 320 × 288 × 2 = **184,320 bytes** (`0x2D000`)             |

### SDRAM / Bridge Address Range

```
Start address : 0x00000000
End address   : 0x0002CFFF  (= 184,320 − 1)
Length        : 0x0002D000  (184,320 bytes)
```

### saved.bin File Format

`saved.bin` is a **headerless raw RGB565 dump** of the framebuffer:
* 184,320 bytes, exactly.
* Row 0 first, left-to-right within each row, pixel pairs packed as described above.
* Written verbatim from SDRAM — what is on-screen is what is in the file.
* `size_maximum: 184320` in `data.json` prevents APF from accepting a file larger than
  one framebuffer.

### Save Operation — Select Button → Target Command 0x0184

When the user presses **Select** (`cont1_key[14]`), the core issues APF Target command
`0x0184` (Data Slot Write) with the following four parameters placed at bridge offset
`0x1020` (default target parameter area):

| Register    | Value        | Meaning                            |
|-------------|--------------|------------------------------------|
| `target_20` | `0x00000022` | Slot ID = 0x22                     |
| `target_24` | `0x00000000` | Slot offset in saved.bin = 0       |
| `target_28` | `0x00000000` | BRIDGE address (SDRAM start)       |
| `target_2C` | `0x0002D000` | Length = 184,320 bytes             |

APF reads 184,320 bytes from bridge `0x00000000`–`0x0002CFFF` and writes them to
`saved.bin` on the SD card, creating or overwriting the file as needed.

Result codes returned in `target_dataslot_err`:

| Code | Meaning                   |
|------|---------------------------|
| 0    | All bytes written OK      |
| 1    | Slot not defined          |
| 2    | Error or out of range     |

### Load Operation — Start Button → Target Command 0x0180

When the user presses **Start** (`cont1_key[15]`), the core issues APF Target command
`0x0180` (Data Slot Read) with the same four parameters.

APF reads `saved.bin` from the SD card and DMA-writes the 184,320 bytes into SDRAM
starting at bridge address `0x00000000`.

Result codes (same register, same values as above).

### Image Bank — A/B/X/Y Buttons → Target Command 0x0180

Buttons B/Y/X/A (`cont1_key[4:7]`) also issue Target command `0x0180` against slot
`0x20` (Image Bank, `ex_image_all.bin`). The slot offset selects one of four
184,320-byte framebuffer images packed back-to-back in the file:

| Button | `cont1_key` bit | Slot offset              |
|--------|-----------------|--------------------------|
| B      | 4               | `0 × 184320 = 0x000000`  |
| Y      | 5               | `1 × 184320 = 0x02D000`  |
| X      | 6               | `2 × 184320 = 0x05A000`  |
| A      | 7               | `3 × 184320 = 0x087000`  |

In all cases the BRIDGE address is `0x00000000` and the length is `184,320` bytes.

### Host/Target Command Bridge Registers Used

| Bridge address   | Direction  | Purpose                                              |
|------------------|------------|------------------------------------------------------|
| `0xF8xx1000`     | Core → APF | Target command/status (write `0x636Mxxxx` to start)  |
| `0xF8xx1020`     | Core → APF | Target parameter word 0 — slot ID                    |
| `0xF8xx1024`     | Core → APF | Target parameter word 1 — slot offset                |
| `0xF8xx1028`     | Core → APF | Target parameter word 2 — BRIDGE address             |
| `0xF8xx102C`     | Core → APF | Target parameter word 3 — length                     |
| `0x2000`–`0x20FF`| APF → Core | Dataslot ID/Size table (32 × 2 words)                |


## Legal
Analogue's Development program was created to further video game hardware preservation with FPGA technology. Analogue does not support or endorse the use of infringing content.

Analogue Developers have access to Analogue Pocket I/O's so Developers can utilize cartridge adapters or interface with other pieces of original or bespoke hardware to support legacy media.
