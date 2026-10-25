# IAC Lab 1 – Counter (notes)

## Big picture
- **SystemVerilog (.sv)** describes the hardware.
- **Verilator** turns the `.sv` into C++, then compiles it together with a **C++ testbench** into an executable (`obj_dir/Vcounter`). Running that executable *is* the simulation.
- Verilator is **cycle-accurate**: it only evaluates once per clock, signals are only 0/1 (no X/Z, no glitches, no gate delays). Fast, and Verilator-clean code is basically synthesizable.
- **Testbench** = wrapper program that creates the DUT, drives inputs (`clk`, `rst`, `en`), and reads outputs (`count`).
- **GTKWave** opens the `.vcd` trace file to view waveforms.
- **Vbuddy** = board (RISC-V MCU + TFT screen + rotary knob + mic…) that lets the simulation drive real I/O over USB serial.

## The counter (`counter.sv`)
```systemverilog
module counter #(parameter WIDTH = 8)(
  input  logic             clk, rst, en,
  output logic [WIDTH-1:0] count
);
always_ff @(posedge clk)
  if (rst) count <= {WIDTH{1'b0}};
  else     count <= count + {{WIDTH-1{1'b0}}, en};
endmodule
```
- File name must match module name.
- `always_ff @(posedge clk)` = clocked (flip-flop) logic. Use non-blocking `<=` inside it.
- `{WIDTH{1'b0}}` = replication → WIDTH zeros.
- `{{WIDTH-1{1'b0}}, en}` = 7 zeros with `en` as the last bit → adds 1 if `en`, 0 otherwise. Written this way so both sides of `+` are exactly WIDTH bits (avoids `-Wall` width warnings).
- Reset here is **synchronous** (only checked on the clock edge). Async version: `always_ff @(posedge clk, posedge rst)`.

## Testbench skeleton (`counter_tb.cpp`)
1. `#include "Vcounter.h"`, `verilated.h`, `verilated_vcd_c.h` (and `vbuddy.cpp` for Task 2)
2. Create DUT: `Vcounter* top = new Vcounter;`
3. Turn on tracing → `tfp->open("counter.vcd");`
4. Set initial inputs (`clk=1, rst=1, en=0`)
5. Loop over cycles: inner loop toggles `clk` twice, calling `tfp->dump()` and `top->eval()` each time
6. Change stimuli after each cycle (e.g. `rst = (i<2)|(i==15)`)
7. `tfp->close(); exit(0);`

## Build & run (`doit.sh`)
```sh
#!/bin/sh
rm -rf obj_dir
rm -f counter.vcd
verilator -Wall --cc --trace counter.sv --exe counter_tb.cpp
make -j -C obj_dir/ -f Vcounter.mk Vcounter
obj_dir/Vcounter
```
Run with `./doit.sh` (after `chmod +x doit.sh`) from inside the task folder.

## Vbuddy (Task 2)
- `vbdOpen()` / `vbdClose()` – connect/disconnect; `vbdHeader("…")` – title on screen
- `vbdHex(digit, value)` – one hex digit on the fake 7-seg display (that's why it counts in hex)
- `vbdPlot(value, min, max)` – plot a point on the TFT (the colour screen) → sawtooth for a counter
- `vbdCycle(n)` – show cycle number
- `vbdFlag()` – state of the knob's push-button (toggles 0/1 each press in default TOGGLE mode). Starts at 0, so with `en = vbdFlag()` nothing moves until you press the knob.
- `vbdValue()` – number set by turning the knob (Task 3)

## Challenge: up/down counter
```systemverilog
else count <= en ? count + 1'b1 : count - 1'b1;
```
- `? :` = 2-to-1 mux; `en` now picks direction instead of on/off.
- 8-bit wraps: `00 - 1 = FF`.

## Setup gotchas I hit (WSL on Windows)
- Run everything in the **VS Code terminal (WSL Ubuntu)**, not PowerShell.
- Vbuddy must be shared into WSL each time it's plugged in / WSL restarts:
  `~/Documents/iac/lab0-devtools/tools/attach_usb.sh`, then `ls /dev/ttyU*` → `/dev/ttyUSB0`.
- `vbuddy.cfg` contains just the port name: `/dev/ttyUSB0`.
- `Error opening port` = permissions → `sudo usermod -aG dialout $USER`, then `wsl --shutdown` (PowerShell) and reopen. Check with `groups`.
- Only lines saying **error** stop the build; the `fgets` warning from `vbuddy.cpp` is harmless.
- VS Code red squiggles on `Vcounter.h` are fine — that header only exists after Verilator runs.
- Git: set `user.name` / `user.email` once; keep `obj_dir/` and `*.vcd` in `.gitignore`; commit after each task.
