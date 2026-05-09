# Typewriter Animation — Full Verilog Design

## Animation State Machine Overview

```
 ┌──────────────────────────────────────────────────────┐
 │                                                      │
 ▼                                                      │
[TYPING]                                                │
  Reveal one character every 10 screen frames           │
  Cursor `_` follows the typing point                   │
  → when char_count == 18, go to PAUSE                  │
     │                                                   │
     ▼                                                   │
[PAUSE]                                                 │
  Full sentence visible, cursor blinks at end           │
  Wait 180 screen frames (3 sec @ 60Hz)                │
  → go to DELETING                                      │
     │                                                   │
     ▼                                                   │
[DELETING]                                              │
  Remove one character every 10 screen frames           │
  Cursor follows the deletion point                     │
  → when char_count == 0, restart TYPING ──────────────┘
```

---

## Text Layout on Screen

- Resolution: 800 × 600 pixels
- Character cell: 8 wide × 16 tall
- Grid: **100 columns × 37 rows**
- Sentence: `"As Above, So Below"` → **18 characters**
- Centered horizontally: `START_COL = (100 - 18) / 2 = 41`
- Centered vertically: `CENTER_ROW = 37 / 2 = 18`

```
Col:  0         41                      58        99
      ·  ·  ·  As Above, So Below_  ·  ·  ·   Row 18
```

---

## Complete Verilog: `ascii_anim.v` (Animation Module)

Drop this into your project. It takes the character grid position as input and outputs the ASCII code to display.

```verilog
`timescale 1ns / 1ps
`default_nettype wire
//------------------------------------------------------------
// Module: ascii_anim
// 
// Typewriter animation for "As Above, So Below"
// 
// Inputs:
//   clk      - Pixel clock (40 MHz)
//   rst      - Synchronous reset
//   vs       - VGA vsync (used to count screen frames)
//   char_x   - Current character column (0-99)
//   char_y   - Current character row (0-36)
//
// Output:
//   ascii_out - ASCII code of character to display at (char_x, char_y)
//------------------------------------------------------------
module ascii_anim (
    input  wire        clk,
    input  wire        rst,
    input  wire        vs,
    input  wire [6:0]  char_x,
    input  wire [5:0]  char_y,
    output wire [6:0]  ascii_out
);

    // --------------------------------------------------------
    // Sentence ROM
    // "As Above, So Below" = 18 characters
    // --------------------------------------------------------
    localparam SENTENCE_LEN = 18;
    
    reg [7:0] sentence [0:SENTENCE_LEN-1];
    initial begin
        sentence[0]  = "A";
        sentence[1]  = "s";
        sentence[2]  = " ";
        sentence[3]  = "A";
        sentence[4]  = "b";
        sentence[5]  = "o";
        sentence[6]  = "v";
        sentence[7]  = "e";
        sentence[8]  = ",";
        sentence[9]  = " ";
        sentence[10] = "S";
        sentence[11] = "o";
        sentence[12] = " ";
        sentence[13] = "B";
        sentence[14] = "e";
        sentence[15] = "l";
        sentence[16] = "o";
        sentence[17] = "w";
    end

    // --------------------------------------------------------
    // Layout constants
    // --------------------------------------------------------
    localparam CENTER_ROW = 6'd18;          // vertical center
    localparam START_COL  = 7'd41;          // (100 - 18) / 2
    localparam END_COL    = 7'd58;          // START_COL + SENTENCE_LEN - 1

    // --------------------------------------------------------
    // Timing constants (all in screen frames, ~60 Hz)
    // --------------------------------------------------------
    localparam FRAMES_PER_CHAR = 8'd10;     // ~167ms per character
    localparam PAUSE_FRAMES    = 8'd180;    // 3 seconds
    localparam BLINK_HALF      = 6'd30;     // cursor toggles every 30 frames (0.5Hz)

    // --------------------------------------------------------
    // State machine
    // --------------------------------------------------------
    localparam STATE_TYPING   = 2'd0;
    localparam STATE_PAUSE    = 2'd1;
    localparam STATE_DELETING = 2'd2;

    reg [1:0]  state;
    reg [4:0]  char_count;    // number of characters currently visible (0-18)
    reg [7:0]  char_timer;    // frames elapsed since last character change
    reg [7:0]  pause_timer;   // frames elapsed in PAUSE state
    reg [5:0]  blink_timer;   // frames since last cursor toggle
    reg        cursor_on;     // 1 = cursor visible, 0 = cursor hidden

    // Detect rising edge of vsync = new screen frame
    reg vs_last;
    wire new_frame = (~vs_last & vs);

    always @(posedge clk) begin
        if (rst) begin
            state       <= STATE_TYPING;
            char_count  <= 5'd0;
            char_timer  <= 8'd0;
            pause_timer <= 8'd0;
            blink_timer <= 6'd0;
            cursor_on   <= 1'b1;
            vs_last     <= 1'b0;
        end else begin
            vs_last <= vs;

            if (new_frame) begin

                // --- Cursor blink (runs in all states) ---
                if (blink_timer == BLINK_HALF - 1) begin
                    blink_timer <= 6'd0;
                    cursor_on   <= ~cursor_on;
                end else begin
                    blink_timer <= blink_timer + 1'b1;
                end

                // --- State machine ---
                case (state)

                    STATE_TYPING: begin
                        if (char_count == SENTENCE_LEN) begin
                            // Finished typing → pause
                            state       <= STATE_PAUSE;
                            pause_timer <= 8'd0;
                            char_timer  <= 8'd0;
                            // Reset cursor so it starts visible during pause
                            cursor_on   <= 1'b1;
                            blink_timer <= 6'd0;
                        end else begin
                            if (char_timer == FRAMES_PER_CHAR - 1) begin
                                char_timer <= 8'd0;
                                char_count <= char_count + 1'b1;
                            end else begin
                                char_timer <= char_timer + 1'b1;
                            end
                        end
                    end

                    STATE_PAUSE: begin
                        if (pause_timer == PAUSE_FRAMES - 1) begin
                            state      <= STATE_DELETING;
                            char_timer <= 8'd0;
                            // Cursor stays on at start of delete
                            cursor_on   <= 1'b1;
                            blink_timer <= 6'd0;
                        end else begin
                            pause_timer <= pause_timer + 1'b1;
                        end
                    end

                    STATE_DELETING: begin
                        if (char_count == 5'd0) begin
                            // Finished deleting → restart
                            state      <= STATE_TYPING;
                            char_timer <= 8'd0;
                            cursor_on  <= 1'b1;
                            blink_timer <= 6'd0;
                        end else begin
                            if (char_timer == FRAMES_PER_CHAR - 1) begin
                                char_timer <= 8'd0;
                                char_count <= char_count - 1'b1;
                            end else begin
                                char_timer <= char_timer + 1'b1;
                            end
                        end
                    end

                    default: state <= STATE_TYPING;
                endcase
            end
        end
    end

    // --------------------------------------------------------
    // Display Logic (combinational)
    // Determine which ASCII character to show at (char_x, char_y)
    // --------------------------------------------------------
    wire is_text_row    = (char_y == CENTER_ROW);
    wire [6:0] col_idx  = char_x - START_COL;        // position within sentence (valid only in range)
    wire in_typed_area  = is_text_row &&
                          (char_x >= START_COL) &&
                          (col_idx < char_count);     // character has been "typed"
    wire is_cursor_pos  = is_text_row &&
                          (char_x == START_COL + char_count) &&
                          (char_x <= END_COL + 1);    // don't let cursor go past screen area

    wire [6:0] char_at_pos = sentence[col_idx[4:0]];  // safe 5-bit index into sentence ROM

    assign ascii_out =
        (is_cursor_pos && cursor_on) ? 7'h5F :          // "_" (underscore, 0x5F)
        in_typed_area                ? char_at_pos :    // the actual letter
                                       7'h20;           // space (0x20)

endmodule
```

---

## Wiring It into `top.v`

Inside your `top.v`, after the timing module:

```verilog
// Character grid position (derived from pixel position)
wire [6:0] char_x = px[9:3];   // px / 8  (px is the pixel column from dvi_timing)
wire [5:0] char_y = py[9:4];   // py / 16 (py is the pixel row from dvi_timing)

// Animation module
wire [6:0] display_char;

ascii_anim anim (
    .clk      (clk_40),
    .rst      (reset),
    .vs       (vs),           // vsync from dvi_timing
    .char_x   (char_x),
    .char_y   (char_y),
    .ascii_out(display_char)
);

// Font renderer
wire pixel_on;

vga_font font (
    .clk       (clk_40),
    .ascii_code(display_char),
    .row       (py[3:0]),     // sub-row within the character cell
    .col       (px[2:0]),     // sub-col within the character cell
    .pixel     (pixel_on)
);

// Color: amber-on-black for that retro terminal feel
wire [7:0] r_out = pixel_on ? 8'hFF : 8'h00;
wire [7:0] g_out = pixel_on ? 8'hB0 : 8'h00;
wire [7:0] b_out = pixel_on ? 8'h00 : 8'h00;
```

---

## Timing Verification

| Event | Calculation | Duration |
|---|---|---|
| Type 1 character | 10 frames × (1/60 Hz) | ~167 ms |
| Type full sentence (18 chars) | 18 × 167 ms | ~3.0 sec |
| Pause | 180 frames × (1/60 Hz) | 3.0 sec exactly |
| Delete full sentence | 18 × 167 ms | ~3.0 sec |
| **Full loop** | | **~9 seconds** |
| Cursor blink rate | toggle every 30 frames | 1 Hz (on 0.5s, off 0.5s) |

---

## Notes for Beginners

- **Why `vs` for timing?** The vsync pulse fires exactly once per screen refresh (~60 times/second). Using it as a "frame tick" is more reliable than counting pixel clocks — it's self-correcting to the actual display rate.
- **Why not use `always @(*)` for `ascii_out`?** The `sentence` array lookup is registered (ROM), but in this design it's a simple reg array that tools will infer as distributed RAM or LUT RAM — fine for 18 bytes.
- **`col_idx[4:0]`** — The 5-bit slice ensures the ROM address is never wider than needed (2⁵=32 > 18 entries).
- **Cursor at `char_count == 0`** — At the very start of each TYPING cycle, the cursor blinks on an empty line. This looks like the terminal is ready and waiting to type, which is a nice subtle touch.
