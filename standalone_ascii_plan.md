# Standalone Plug-and-Play ASCII Art Animation on FPGA

## The Core Idea Shift

The existing `vga_to_ascii` project is a **pass-through converter** — it *receives* a VGA signal from a PC, processes it, and outputs it. You want almost the opposite: a **self-contained display** that *generates* its own video signal and plays an animation stored inside the FPGA. No host PC needed.

Think of it like the difference between a security camera monitor (pass-through) and a digital photo frame (standalone).

---

## What Changes vs. the Existing Project

| Aspect | Original Project | Your New Project |
|--------|-----------------|-----------------|
| Video signal | **Received** from a host via VGA-in | **Generated** by the FPGA itself |
| Pixel clock | Comes from the external VGA source | Comes from the on-board PLL |
| Image content | Live video frames from a PC | Frames stored in ROM inside the FPGA |
| `top.v` | Complex: VGA capture + averaging | Simple: timing + ROM lookup + render |
| `ascii_lut.v` | Needed (brightness → character) | **NOT needed** (you pick the characters directly) |
| `char_weight.c` | Development tool | **NOT needed** |
| Everything else | Needed | **Mostly reused as-is** |

---

## Files You Can Reuse Directly (Zero Changes)

These are solid, tested hardware modules. Drop them straight into your new project:

| File | Why You Keep It |
|------|----------------|
| `vga_font.v` | ⭐ The font ROM. Give it an ASCII code + row + col → get a pixel. This is your biggest reuse. |
| `dvi_timing.v` | ⭐ Generates HSYNC, VSYNC, and pixel X/Y coordinates for an 800×600 display. This replaces the VGA input entirely. |
| `dvi_module.v` | Sends the final pixel data out to the monitor over DVI. |
| `iic_init.v` | Initializes the DVI chip via I²C. Required for the monitor to accept the signal. |
| `debounce_rst.v` | Cleans up the reset button. Keep it. |
| `x2_dcm.v` | Clock management. Keep it. |
| `ddr2_idelay_ctrl.v` / `_mod.v` | Required by Xilinx tools for I/O timing. Keep it. |
| `constraints.ucf` | Pin assignments for the ML505 board. Keep it. |
| `pll_arwz.ucf` | PLL timing constraints. Keep it. |

## Files You REPLACE

| File | Action |
|------|--------|
| `top.v` | **Rewrite** — remove all VGA-in capture logic; add animation ROM and character lookup |
| `ascii_lut.v` | **Delete** — you don't need to map brightness to characters anymore |

---

## How `dvi_timing.v` Works (This is Your New "Heart")

This file already exists and already does exactly what you need. It generates:
- `hs` — Horizontal sync pulse
- `vs` — Vertical sync pulse  
- `x` — Current pixel column (0–799)
- `y` — Current pixel row (0–599)
- `enable` — Goes high only during the visible part of the screen

At 800×600 resolution you need a **40 MHz pixel clock**. The board's 33 MHz oscillator feeds into the PLL to produce this.

---

## The New Architecture

```
Board Clock (33MHz)
     │
     ▼
[PLL] ─────────────────────────────────────────► 40 MHz pixel clock
     │                                                    │
     ▼                                                    ▼
[debounce_rst]                               [dvi_timing]
     │                                        │   │   │
  reset                                      hs  vs  x, y, enable
                                              │
                               ┌─────────────┘
                               │  char_x = x / 8   (which column of text)
                               │  char_y = y / 16  (which row of text)
                               │  sub_col = x[2:0] (pixel within character, 0-7)
                               │  sub_row = y[3:0] (pixel within character, 0-15)
                               │
                               ▼
                    [Animation Frame ROM]
                     address = frame * (100*37) + char_y * 100 + char_x
                     output  = ASCII character code (7 bits)
                               │
                               ▼
                    [vga_font] ◄── sub_col, sub_row
                     output = pixel (1 or 0)
                               │
                               ▼
                    [Color Logic]
                     if pixel=1 → foreground color (e.g., bright green)
                     if pixel=0 → background color (e.g., black)
                               │
                               ▼
                    [dvi_module] → Monitor
```

---

## Understanding the Character Grid

At 800×600 with 8×16-pixel characters:
- **100 characters wide** (800 ÷ 8)
- **37 characters tall** (600 ÷ 16 = 37.5, truncated to 37)

Each **frame** of your animation is a 100×37 grid of ASCII character codes = **3,700 bytes per frame**.

In Verilog, you store this as a ROM initialized with `$readmemh` or hardcoded `initial` values.

---

## How Animation Works: Frame Timing

You need a **frame counter** that advances slowly enough to see each frame. At 40 MHz pixel clock and 800×600:

- One full frame takes: `1056 * 628 = ~663,000 clock cycles` (including blanking)
- At 40 MHz, that's ~60 frames per second of screen refresh

To animate at e.g. **4 animation frames per second**, you hold each animation frame for 15 screen refreshes:

```verilog
reg [3:0]  anim_frame;     // which animation frame (0-N)
reg [3:0]  frame_hold;     // how many screen refreshes we've shown this frame
reg        vs_last;

always @(posedge clk) begin
    vs_last <= vs;
    if (vs_last == 0 && vs == 1) begin  // rising edge of vsync = new screen frame
        if (frame_hold == 14) begin
            frame_hold  <= 0;
            anim_frame  <= anim_frame + 1;
        end else begin
            frame_hold <= frame_hold + 1;
        end
    end
end
```

---

## How to Store Your Animation (ROM Design)

### Option A: Hardcode in Verilog (simplest for beginners)

```verilog
// A 2-frame animation with a 4×3 character "screen" (tiny example)
// In real life: 100×37 = 3700 chars per frame
reg [6:0] anim_rom [0:1][0:11]; // 2 frames, 12 chars each

initial begin
    // Frame 0: "Hello World!"
    anim_rom[0][0]  = "H"; anim_rom[0][1]  = "e";
    anim_rom[0][2]  = "l"; anim_rom[0][3]  = "l";
    // ... fill all 3700 per frame
    
    // Frame 1: slightly shifted
    // ...
end
```

### Option B: Use $readmemh with a text file (recommended for many frames)

```verilog
reg [6:0] anim_rom [0:NUM_FRAMES * 3700 - 1];

initial begin
    $readmemh("animation.hex", anim_rom);
end
```

Then generate `animation.hex` with a Python script on your PC before synthesis.

### Option C: Block RAM (BRAM) — for large animations

Xilinx FPGAs have dedicated Block RAM tiles. `vga_font.v` already uses one (`RAMB16_S1`). You can instantiate another BRAM to hold your animation frames.

---

## A Worked Example: Simple "Bouncing Block" Animation

Imagine a `@` character bouncing around the screen — like the DVD logo but made of ASCII.

**Animation data approach:**
- The ROM only needs to store **the whole screen** for each frame
- Most characters are `space`; only the bouncing `@` position changes

**Alternatively** (simpler in hardware): don't use a ROM at all. **Calculate** the character on the fly:

```verilog
// Ball position updated each vsync
reg [6:0] ball_x;   // 0-99 (character column)
reg [5:0] ball_y;   // 0-36 (character row)
reg       dx, dy;   // direction

always @(posedge clk) begin
    if (new_vsync_frame) begin
        // Move ball
        ball_x <= ball_x + (dx ? 1 : -1);
        ball_y <= ball_y + (dy ? 1 : -1);
        // Bounce off walls
        if (ball_x == 0 || ball_x == 99) dx <= ~dx;
        if (ball_y == 0 || ball_y == 36) dy <= ~dy;
    end
end

// What character to show at (char_x, char_y)?
wire [6:0] display_char = (char_x == ball_x && char_y == ball_y) ? "@" : " ";
```

**No ROM needed at all for this pattern!** This is the most FPGA-efficient approach for procedural animations.

---

## Your New `top.v` Skeleton

Here is what your new top module looks like (compared to the old 254-line monster):

```verilog
module top (
    // System
    input  CLK_33MHZ_FPGA,
    input  FPGA_CPU_RESET_B,
    // DVI output (same pins as before)
    output [11:0] DVI_D,
    output DVI_DE, DVI_H, DVI_V,
    output DVI_RESET_B, DVI_XCLK_N, DVI_XCLK_P,
    inout  IIC_SCL_VIDEO, IIC_SDA_VIDEO,
    input  DVI_GPIO1
);

wire clk_40, clk_33, reset, locked_pll;
// ... PLL, reset debounce (reuse existing modules) ...

// 1. Generate VGA timing
wire [10:0] px, py;
wire hs, vs, enable;
dvi_timing timing (
    .clk(clk_40), .rst(reset),
    .hs(hs), .vs(vs),
    .x(px), .y(py),
    .enable(enable)
);

// 2. Derive character grid position
wire [6:0] char_x = px[9:3];  // px / 8
wire [5:0] char_y = py[9:4];  // py / 16
wire [2:0] sub_col = px[2:0];
wire [3:0] sub_row = py[3:0];

// 3. Determine which character to show (your animation logic here)
wire [6:0] display_char = /* your logic: ROM lookup or procedural */;

// 4. Render the character to a pixel
wire pixel;
vga_font font (
    .clk(clk_40),
    .ascii_code(display_char),
    .row(sub_row),
    .col(sub_col),
    .pixel(pixel)
);

// 5. Choose color
wire [7:0] r = pixel ? 8'h00 : 8'h00;  // black background
wire [7:0] g = pixel ? 8'hFF : 8'h00;  // bright green text
wire [7:0] b = pixel ? 8'h00 : 8'h00;

// 6. Output
dvi_module dvi (
    .pixel_clk(clk_40), .gpuclk_rst(reset),
    .hsync(~hs), .vsync(~vs), .blank_b(enable),
    .pixel_r(r), .pixel_g(g), .pixel_b(b),
    .dvi_vs(DVI_V), .dvi_hs(DVI_H),
    .dvi_d(DVI_D), .dvi_xclk_p(DVI_XCLK_P),
    .dvi_xclk_n(DVI_XCLK_N), .dvi_de(DVI_DE),
    .dvi_reset_b(DVI_RESET_B),
    .dvi_sda(IIC_SDA_VIDEO), .dvi_scl(IIC_SCL_VIDEO),
    .iic_done()
);

endmodule
```

---

## Clock Warning: 33 MHz → 40 MHz

`dvi_timing.v` is parameterized for 800×600 @ 60Hz which needs a **40 MHz pixel clock**. The board has 33 MHz. The existing PLL in `ipcore_dir/` is configured to output 100 MHz (for DDR2 memory). You will need to **reconfigure the PLL** (via Xilinx IP core editor) to output 40 MHz instead of 100 MHz, or add a second PLL output.

Alternatively, you can keep the 33 MHz clock and retune `dvi_timing.v`'s parameters to match — but 33 MHz doesn't hit a standard VGA mode cleanly. Using the PLL at 40 MHz is the right approach.

---

## Step-by-Step Build Roadmap

| Step | What to Do |
|------|-----------|
| **1** | Open the Xilinx ISE project (`.xise`). Delete `top.v` and `ascii_lut.v` from the project (don't delete the files yet). |
| **2** | Reconfigure the PLL IP core to output 40 MHz on CLKOUT0. |
| **3** | Create a new `top.v` from the skeleton above, starting with a procedural animation (e.g., the bouncing `@`). |
| **4** | Synthesize and check for timing/pin errors. |
| **5** | Generate the bitstream and program the FPGA. |
| **6** | Once it works, swap in a ROM-based animation for richer content. |

---

## Animation Ideas Ranked by Complexity

| Idea | Complexity | Approach |
|------|-----------|----------|
| Scrolling text ticker | 🟢 Easy | Shift a string's characters left each frame |
| Bouncing character (`@` / `*`) | 🟢 Easy | Procedural: compute position from counters |
| Spinning ASCII shape | 🟡 Medium | Precompute rotation frames in a ROM |
| Rain effect (Matrix-style) | 🟡 Medium | Per-column random drop counter |
| ASCII fire effect | 🔴 Hard | Cellular automaton in Block RAM |
| Stored animation (sprite) | 🟡 Medium | ROM with Python-generated `.hex` file |

---

## Key Takeaway

The most valuable things the existing project gives you are:
1. **`vga_font.v`** — A complete, working 8×16 bitmap font ROM. This alone saves many hours.
2. **`dvi_timing.v`** — A complete VGA timing generator. Plug in a clock → get x, y, hs, vs for free.
3. **`dvi_module.v`** + **`iic_init.v`** — DVI output stack, ready to go.
4. **`constraints.ucf`** — The pin map for the ML505 board.

Everything else in `top.v` (the VGA capture, averaging, double buffering) gets thrown away and replaced with your much simpler animation logic.
