# Realtime VGA to ASCII Art Converter — Project Report

## What Is This Project?

This is a **hardware-based, realtime VGA to ASCII Art Converter**, built by engineer **Wenting Zhang** and designed to run on a **Xilinx ML505 FPGA** development board.

In plain English: plug a VGA video signal into the board, and instead of displaying the video normally, the FPGA converts every frame in real-time into ASCII art — replacing every small block of the image with a text character like `@`, `M`, `#`, `.`, or `space`. The output is sent to a monitor over DVI.

> Reference: https://hackaday.io/project/66319-realtime-vga-ascii-art-converter

---

## Big Picture: How Does It Work?

The core idea behind ASCII art conversion is simple: **dark areas → sparse characters, bright areas → dense characters.**

The FPGA implements this pipeline in hardware:

```
VGA Signal In
     │
     ▼
[Pixel Clock sync] ─► Count X/Y position of each pixel
     │
     ▼
[Average Block Color] ─► Sum R+G+B for every 8×16 pixel block
     │
     ▼
[Brightness Score] ─► (R+G+B) / 16 → a value 0–47
     │
     ▼
[ASCII LUT] ─► Look up which character matches the brightness
     │
     ▼
[Font ROM] ─► Render that character pixel-by-pixel on screen
     │
     ▼
DVI Signal Out → Monitor
```

---

## Step-by-Step Explanation

### Step 1: Receiving the VGA Signal (`top.v`)

The FPGA receives three 8-bit color channels — **Red, Green, Blue** — along with **Horizontal Sync (HSYNC)** and **Vertical Sync (VSYNC)** signals from the incoming VGA source. These sync signals tell the hardware when a new line starts and when a new frame starts.

The pixel clock (`VGA_IN_DATA_CLK`) drives the entire process.

### Step 2: Dividing the Image into Character Blocks

The incoming 800×600 resolution image is mentally carved up into a grid of **8-pixel wide × 16-pixel tall** cells. Each cell will eventually show one ASCII character.

- 800 ÷ 8 = **100 characters per row**
- 600 ÷ 16 = **37.5 rows** (approx. 37 rows of text)

### Step 3: Averaging Pixel Colors (Double-Buffered)

As pixels stream in, the FPGA **accumulates the sum of R, G, B values** for each column slot (100 slots) in the current row of character cells.

It uses **double buffering** — two sets of accumulators (`avg_r_a`/`avg_r_b`, etc.) that swap after every 16 rows. This way, one buffer is being *written* (accumulating the current row) while the other is being *read* (used to display the previous row). No glitches!

### Step 4: Mapping Brightness to an ASCII Character (`ascii_lut.v`)

The average brightness of a block is computed as:

```verilog
wire [5:0] char_sel = (avg_r_out + avg_g_out + avg_b_out) / 16;
```

This gives a 6-bit value (0–47). `ascii_lut.v` is a simple lookup table mapping each ID to a character, ordered from sparse (dark) to dense (bright):

| ID Range | Characters | Meaning |
|----------|------------|---------|
| 0–3 | `space . \` - ` | Very dark / empty |
| 4–15 | `, : ; ~ + / = > \| ( ) \` | Light marks |
| 16–30 | `i % { * s v 7 a e C J L T Y w` | Medium |
| 31–47 | `F 9 V G X A E $ & # @ R W 0 N M Q` | Very bright / dense |

### Step 5: How the Character Order Was Determined (`char_weight.c`)

This is a standalone **C program** (runs on your PC, not the FPGA). It contains the raw pixel bitmaps for every printable ASCII character and counts how many pixels are "on" in each one (its **visual weight**). Running it sorts all characters from lightest to heaviest — that's how the author decided the ordering in `ascii_lut.v`.

### Step 6: Drawing the Character on Screen (`vga_font.v`)

Once the character is selected, `vga_font.v` acts as a **Font ROM**. Given the ASCII code of the character, and the current pixel row (0–15) and column (0–7) within the 8×16 cell, it outputs a single `1` or `0` — pixel on or off.

### Step 7: Color and Output Mode (DIP Switches)

Physical switches on the FPGA board control the display mode:

| Switch | Function |
|--------|----------|
| `DIP_SW[0]` OFF | Text drawn in **white** |
| `DIP_SW[0]` ON | Text drawn in the **original average color** of that block |
| `DIP_SW[1]` OFF | Pass-through mode — show the **original video** |
| `DIP_SW[1]` ON | **ASCII art mode** — show the converted text |

### Step 8: Sending the Output (`dvi_module.v`)

The final pixel colors are sent out through the **DVI transmitter** using DDR (Double Data Rate) output registers — a Xilinx FPGA primitive that efficiently serializes data on both the rising and falling edges of the clock. An IIC (I²C) initialization sequence configures the DVI chip on startup.

---

## File Map

```
vga_to_ascii/
├── README.md                ← Very brief project description
└── vga_input/
    ├── top.v                ← ⭐ Main module. Connects everything. Core logic lives here.
    ├── ascii_lut.v          ← ⭐ Brightness-to-character lookup table (48 entries)
    ├── vga_font.v           ← ⭐ Font ROM: renders any ASCII character pixel-by-pixel
    ├── char_weight.c        ← 🔧 Helper C tool (run on PC) to rank characters by visual weight
    ├── dvi_module.v         ← Handles DVI/HDMI output signal formatting
    ├── dvi_timing.v         ← DVI timing helpers
    ├── iic_init.v           ← I²C initialization for DVI chip
    ├── x2_dcm.v             ← Clock doubler (Digital Clock Manager)
    ├── ddr2_idelay_ctrl.v   ← I/O delay control (needed by Xilinx tools)
    ├── ddr2_idelay_ctrl_mod.v
    ├── debounce_rst.v       ← Cleans up noisy reset button signal
    ├── test.v               ← Testbench for simulation
    ├── constraints.ucf      ← Pin assignments: maps signal names → physical FPGA pins
    ├── pll_arwz.ucf         ← PLL timing constraints
    ├── vga_input.xise       ← Xilinx ISE project file
    └── ipcore_dir/          ← Auto-generated Xilinx IP cores (PLL, etc.)
```

---

## Getting Started as a Beginner

### What You Need to Know First

1. **Verilog** — The hardware description language used here. Unlike Python or C, Verilog describes *circuits* that all run simultaneously. The key mental shift is:
   - `always @(posedge clk)` = "on every rising edge of the clock, do this"
   - Wires carry signals; `reg` stores values in flip-flops

2. **FPGA Basics** — An FPGA is a chip full of reconfigurable logic. You "program" it by telling it what circuit to become. No CPU is involved; all the logic runs in parallel hardware.

3. **VGA Protocol** — VGA sends pixels one at a time, left-to-right, top-to-bottom. HSYNC pulses at the end of each row; VSYNC pulses at the end of each frame.

### Suggested Learning Path

| Step | What to do |
|------|-----------|
| 1 | Read `ascii_lut.v` — simplest file, just a big case statement |
| 2 | Compile and run `char_weight.c` on your PC to see how the character order was derived |
| 3 | Read `top.v` with focus on the accumulator logic (lines 118–179) |
| 4 | Look at `vga_font.v` to understand how bitmap fonts are stored and accessed |
| 5 | Modify `ascii_lut.v` — e.g., use only `0` and `1` for a binary art effect |

### Quick Experiment Ideas

- **Change the character palette**: Edit `ascii_lut.v` to use emoji-like dense characters or only digits
- **Change the cell size**: Modify the divisor in `avg_addr` (currently `/8`) to make coarser or finer ASCII cells
- **Simulate it**: Open the Xilinx ISE project (`vga_input.xise`) and run `test.v` in ISim to observe signals without any hardware
