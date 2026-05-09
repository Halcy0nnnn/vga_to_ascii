# Learning: animation_engine.v — Typewriter State Machine

## Overview

`animation_engine.v` is the **brain** of your animation. It:
- Tracks animation state (WRITING, HOLDING, ERASING)
- Counts frames and characters
- Manages the blinking cursor
- Outputs the ASCII character that should be displayed at each moment

**Think of it like a VCR controller:**
- The `dvi_timing` module is the VCR playing frames continuously (60 fps)
- The `animation_engine` is the "program guide" saying "frame 0–11: char 0", "frame 12–23: char 1", etc.

---

## State Diagram

```
WRITING (4.2 sec)
  Every 12 frames: advance char_position (0 → 1 → 2 → ... → 21)
  
         (char_position reaches 21)
         │
         ▼
HOLDING (3 sec)
  char_position stays at 21
  Just count down timer
  
         (timer expires)
         │
         ▼
ERASING (4.2 sec)
  Every 12 frames: decrement char_position (21 → 20 → ... → 0)
  
         (char_position reaches 0)
         │
         ▼
[Loop back to WRITING]
```

---

## Verilog Building Blocks You Need

### 1. Module Declaration

```verilog
module animation_engine (
  input clk,              // 50 MHz clock
  input reset,            // Active high reset
  output [7:0] ascii_code,    // Current ASCII character
  output cursor_visible       // Is cursor blinking on?
);
```

**Key Points:**
- `input clk` — tied to 50 MHz from PLL
- `reset` — when asserted, module should return to initial state
- `ascii_code` — output the ASCII code of the character to display
- `cursor_visible` — 1 = show cursor, 0 = hide cursor

### 2. Parameter Declaration

```verilog
// Define constants so they're easy to change later
parameter [7:0] CHAR_A = 8'd65;           // ASCII 'A'
parameter [7:0] CHAR_S = 8'd83;           // ASCII 'S'
parameter [7:0] CHAR_SPACE = 8'd32;       // ASCII ' '
parameter [7:0] CHAR_UNDERSCORE = 8'd95;  // ASCII '_'

parameter FRAMES_PER_CHAR = 12;            // Frames between char advances
parameter HOLD_FRAMES = 180;               // Frames to hold (3 sec × 60 fps)
parameter CURSOR_BLINK_FRAMES = 30;        // Frames on/off for cursor
parameter MAX_CHAR_POS = 21;               // Length of "As Above, So Below"
```

### 3. Registers (State Variables)

```verilog
// State machine: which phase are we in?
reg [1:0] state;  // 0=WRITING, 1=HOLDING, 2=ERASING
localparam WRITING = 2'd0;
localparam HOLDING = 2'd1;
localparam ERASING = 2'd2;

// Counters
reg [5:0] frame_counter;       // 0–59 (which frame out of 60 in current second)
reg [3:0] char_frame_counter;  // 0–11 (which frame out of 12 for current character)
reg [4:0] char_position;       // 0–21 (which character are we on?)
reg [7:0] hold_counter;        // 0–179 (how many frames held?)
reg [5:0] blink_counter;       // 0–59 (for cursor blink timing)

// The text we're displaying
reg [7:0] text_rom [0:20];  // Array of 21 ASCII bytes
```

### 4. ROM Initialization

```verilog
initial begin
  // "As Above, So Below"
  text_rom[0]  = 8'd65;  // 'A'
  text_rom[1]  = 8'd115; // 's'
  text_rom[2]  = 8'd32;  // ' '
  text_rom[3]  = 8'd65;  // 'A'
  text_rom[4]  = 8'd98;  // 'b'
  text_rom[5]  = 8'd111; // 'o'
  text_rom[6]  = 8'd118; // 'v'
  text_rom[7]  = 8'd101; // 'e'
  text_rom[8]  = 8'd44;  // ','
  text_rom[9]  = 8'd32;  // ' '
  text_rom[10] = 8'd83;  // 'S'
  text_rom[11] = 8'd111; // 'o'
  text_rom[12] = 8'd32;  // ' '
  text_rom[13] = 8'd66;  // 'B'
  text_rom[14] = 8'd101; // 'e'
  text_rom[15] = 8'd108; // 'l'
  text_rom[16] = 8'd111; // 'o'
  text_rom[17] = 8'd119; // 'w'
end
```

### 5. Sequential Logic (Main State Machine)

```verilog
always @(posedge clk) begin
  if (reset) begin
    // Reset everything
    state <= WRITING;
    frame_counter <= 0;
    char_frame_counter <= 0;
    char_position <= 0;
    hold_counter <= 0;
    blink_counter <= 0;
  end else begin
    // Increment frame counter every clock cycle
    frame_counter <= frame_counter + 1;
    
    // Every 60 cycles, increment character frame counter
    if (frame_counter == 59) begin
      frame_counter <= 0;
      char_frame_counter <= char_frame_counter + 1;
      blink_counter <= blink_counter + 1;
      
      // Every 12 character frames, do something (depends on state)
      if (char_frame_counter == 11) begin  // Will be 11 before rolling over
        char_frame_counter <= 0;
        
        case (state)
          WRITING: begin
            // Advance to next character
            if (char_position < MAX_CHAR_POS) begin
              char_position <= char_position + 1;
            end else begin
              // Finished writing all 21 characters
              state <= HOLDING;
              hold_counter <= 0;
            end
          end
          
          HOLDING: begin
            // Just count down the hold timer
            hold_counter <= hold_counter + 1;
            if (hold_counter == HOLD_FRAMES - 1) begin
              // Hold time expired
              state <= ERASING;
              char_position <= MAX_CHAR_POS - 1;  // Start from last char
            end
          end
          
          ERASING: begin
            // Erase character by character
            if (char_position > 0) begin
              char_position <= char_position - 1;
            end else begin
              // Finished erasing, back to writing
              state <= WRITING;
            end
          end
        endcase
      end
    end
  end
end
```

### 6. Combinational Logic (Output Generation)

```verilog
// Output current ASCII code based on character position
assign ascii_code = text_rom[char_position];

// Cursor blinks at ~1 Hz (on for 30 frames, off for 30 frames)
assign cursor_visible = (blink_counter < 30) ? 1 : 0;
```

---

## Complete Skeleton Module

Here's a minimal working version:

```verilog
`timescale 1ns / 1ps

module animation_engine (
  input clk,
  input reset,
  output [7:0] ascii_code,
  output cursor_visible
);

  // State constants
  localparam WRITING = 2'd0;
  localparam HOLDING = 2'd1;
  localparam ERASING = 2'd2;
  
  // Timing constants
  localparam FRAMES_PER_CHAR = 12;
  localparam HOLD_FRAMES = 180;
  localparam MAX_CHAR_POS = 21;

  // State variables
  reg [1:0] state;
  reg [5:0] frame_counter;
  reg [3:0] char_frame_counter;
  reg [4:0] char_position;
  reg [7:0] hold_counter;
  reg [5:0] blink_counter;

  // Text storage
  reg [7:0] text_rom [0:20];

  initial begin
    // Initialize text: "As Above, So Below"
    text_rom[0]  = "A";
    text_rom[1]  = "s";
    text_rom[2]  = " ";
    text_rom[3]  = "A";
    text_rom[4]  = "b";
    text_rom[5]  = "o";
    text_rom[6]  = "v";
    text_rom[7]  = "e";
    text_rom[8]  = ",";
    text_rom[9]  = " ";
    text_rom[10] = "S";
    text_rom[11] = "o";
    text_rom[12] = " ";
    text_rom[13] = "B";
    text_rom[14] = "e";
    text_rom[15] = "l";
    text_rom[16] = "o";
    text_rom[17] = "w";
  end

  // Main state machine
  always @(posedge clk) begin
    if (reset) begin
      state <= WRITING;
      frame_counter <= 0;
      char_frame_counter <= 0;
      char_position <= 0;
      hold_counter <= 0;
      blink_counter <= 0;
    end else begin
      frame_counter <= frame_counter + 1;
      
      if (frame_counter == 59) begin
        frame_counter <= 0;
        char_frame_counter <= char_frame_counter + 1;
        blink_counter <= blink_counter + 1;
        
        if (char_frame_counter == 11) begin
          char_frame_counter <= 0;
          
          case (state)
            WRITING: begin
              if (char_position < MAX_CHAR_POS) begin
                char_position <= char_position + 1;
              end else begin
                state <= HOLDING;
                hold_counter <= 0;
              end
            end
            
            HOLDING: begin
              hold_counter <= hold_counter + 1;
              if (hold_counter == HOLD_FRAMES - 1) begin
                state <= ERASING;
                char_position <= MAX_CHAR_POS - 1;
              end
            end
            
            ERASING: begin
              if (char_position > 0) begin
                char_position <= char_position - 1;
              end else begin
                state <= WRITING;
              end
            end
          endcase
        end
      end
    end
  end

  // Output assignments
  assign ascii_code = text_rom[char_position];
  assign cursor_visible = (blink_counter < 30) ? 1 : 0;

endmodule
```

---

## Key Verilog Concepts Explained

### 1. `always @(posedge clk)`
**What it means:** Run this block every time the clock rises (goes from 0 to 1).

**Why use it:** Synchronous logic. All your state changes happen at well-defined clock edges, not randomly.

### 2. `if (reset)`
**What it means:** If the reset signal is high, set everything to initial values.

**Why important:** Always have a way to initialize your state machine. Without it, after power-up, registers have random values and the FSM breaks.

### 3. `reg` vs `wire`
- **`reg`**: A storage element (like a latch or flip-flop). Retains its value between clock cycles. Used for state variables.
- **`wire`**: A combinational connection (like a logic gate output). Changes immediately based on inputs. Used for outputs calculated from regs.

### 4. `case (state)`
**What it means:** A switch statement. Do different things depending on which state you're in.

```verilog
case (state)
  WRITING:  // if state == WRITING, do this
    char_position <= char_position + 1;
  HOLDING:  // if state == HOLDING, do this
    hold_counter <= hold_counter + 1;
  ERASING:  // if state == ERASING, do this
    char_position <= char_position - 1;
endcase
```

### 5. `assign` (Combinational Logic)
**What it means:** Continuously evaluate the right side and drive the output.

```verilog
assign cursor_visible = (blink_counter < 30) ? 1 : 0;
// If blink_counter < 30, cursor_visible = 1; else 0
// This happens immediately, no clock needed
```

### 6. Non-blocking Assignment `<=`
**What it means:** In sequential blocks, use `<=` (not `=`) to avoid simulation race conditions.

```verilog
always @(posedge clk) begin
  // Use non-blocking (<=)
  frame_counter <= frame_counter + 1;  ✓ Correct
  
  // NOT blocking (=) — can cause simulation bugs
  // frame_counter = frame_counter + 1;  ✗ Wrong
end
```

---

## Timing Walkthrough (Example)

Let's trace what happens frame-by-frame:

```
Frame #0 (t=0): 
  - clk rises
  - frame_counter: 0 → 1
  - char_frame_counter: 0 (no change yet)
  - char_position: 0
  - state: WRITING
  - Output: ascii_code = text_rom[0] = 'A'

Frame #1–10:
  - frame_counter: 1→2→3→...→11
  - (No state changes yet)
  - Output: ascii_code = 'A' (still)

Frame #11:
  - frame_counter: 11 → 12... (reaches 59 next, not yet)

Frame #59:
  - frame_counter reaches 59
  - frame_counter: 59 → 0 (resets)
  - char_frame_counter: 0 → 1 (increments)
  - blink_counter: 0 → 1
  - char_frame_counter is 1, NOT 11, so no state change yet
  - Output: ascii_code = 'A'

Frame #60:
  - frame_counter: 0 → 1
  - (clock cycles through again)

...

Frame #719 (at 60 frames/sec, this is ~12 seconds in):
  - frame_counter reaches 59 (for the 12th time)
  - frame_counter: 59 → 0
  - char_frame_counter: 11 (has been counting: 0,1,2,...,11 = 12 values)
  - NOW: char_frame_counter == 11 is TRUE
  - Action: char_frame_counter <= 0, char_position <= 1
  - Output: ascii_code = text_rom[1] = 's'

Frame #720:
  - Output: ascii_code = 's' (now showing second character)
```

---

## Testing Your Logic (Thought Exercise)

**Question**: If you wanted to change the speed to 8 frames per character instead of 12, what would you change?

**Answer**: Just change the parameter:
```verilog
localparam FRAMES_PER_CHAR = 8;
```

And update the comparison in the state machine:
```verilog
if (char_frame_counter == FRAMES_PER_CHAR - 1) begin
  char_frame_counter <= 0;
  // ... do state transitions ...
end
```

**Question**: What if the cursor blink is too slow? How would you make it blink faster?

**Answer**: Change the cursor threshold:
```verilog
// Blink twice per second (30 frames per half-second × 2 half-seconds = 60 frames per full cycle)
assign cursor_visible = (blink_counter < 15) ? 1 : 0;  // Faster: 15 frames on/off
```

---

## Common Mistakes to Avoid

1. **Using `=` instead of `<=` in sequential blocks**
   - ❌ `frame_counter = frame_counter + 1;`
   - ✓ `frame_counter <= frame_counter + 1;`

2. **Forgetting the reset condition**
   - Without `if (reset)`, your FSM might not initialize properly after power-up

3. **Off-by-one errors with counters**
   - Remember: if you want a counter to reach 11, check `if (counter == 11)`, and it's 0–11 = 12 states total
   - If you want 12 frames per char: check `char_frame_counter == 11` (since it goes 0–11)

4. **Not resetting sub-counters when changing state**
   - When transitioning from WRITING to HOLDING, remember to reset `char_frame_counter <= 0` and `frame_counter <= 0`

5. **Mixing combinational and sequential logic incorrectly**
   - Outputs like `ascii_code` should be `assign`, not inside `always @(posedge clk)`

---

## Next: Building the Rest

Once you're comfortable with this, we can:

1. **Create a testbench** — simulate the state machine and verify timing
2. **Write `char_compositor.v`** — take these ASCII codes and render them on screen
3. **Integrate into `top.v`** — wire everything together
4. **Test on hardware** — program the FPGA and see it work!

---

## Reference: ASCII Codes You'll Need

```
'A' = 65     'S' = 83     'B' = 66     'e' = 101
's' = 115    'o' = 111    'b' = 98     'l' = 108
'v' = 118    'w' = 119    ',' = 44     ' ' = 32
'_' = 95     (cursor underscore)
```

Or in Verilog, just use the character directly:
```verilog
text_rom[0] = "A";  // Verilog interprets this as ASCII 65
text_rom[1] = "s";  // ASCII 115
```
