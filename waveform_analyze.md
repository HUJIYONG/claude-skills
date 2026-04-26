# Skill: Analyzing VCD Waveforms with waveform-cli
## Tool Reference
```
waveform-cli - Command-line interface for waveform analysis

Usage: waveform-cli <command1> [args...] [-- <command2> [args...] ...]

Commands can be chained using '--' as a separator.

Commands:
  open_waveform <file_path> [--alias <alias>]
    Open a VCD or FST waveform file

  close_waveform <waveform_id>
    Close a waveform and free its memory

  list_signals <waveform_id> [--pattern <pattern>] [--hierarchy <prefix>] [--recursive <true|false>] [--limit <n>]
    List signals matching optional pattern

  read_signal <waveform_id> <signal_path> [--time-index <idx> | --time-indices <idx1,idx2,...>]
    Read signal values at specific time indices

  get_signal_info <waveform_id> <signal_path>
    Get metadata about a signal

  find_signal_events <waveform_id> <signal_path> [--start <idx>] [--end <idx>] [--limit <n>]
    Find all changes (events) of a signal in a time range

  find_conditional_events <waveform_id> <condition> [--start <idx>] [--end <idx>] [--limit <n>]
    Find events where a condition is satisfied
    Condition syntax supports: signal paths, ~ (NOT), & (AND), | (OR),
    ^ (XOR), &&, ||, ==, !=, $past(), bit extraction, Verilog literals

Examples:
  waveform-cli open_waveform test.vcd
  waveform-cli open_waveform test.fst --alias mywave -- list_signals mywave --pattern clock
  waveform-cli open_waveform test.vcd -- read_signal test top.clk --time-index 0
  waveform-cli open_waveform test.vcd -- find_signal_events test top.reset --limit 10
```

> **Note:** In bash, `$` in condition strings must be escaped as `\$` (e.g. `\$past()`).
---
## Techniques
### 1. Get All Events of a Signal
Use `find_signal_events` to list every value change of a signal. Useful for control signals such as reset, enable, or valid.
```bash
waveform-cli open_waveform foo.vcd --alias w \
  -- find_signal_events w TOP.rst_n --limit 50
```
### 2. Read Signal Values at Specific Time Indices
Use `read_signal` with `--time-indices` to sample one or more signals at a known set of time points. Multiple signals can be read in the same invocation by chaining commands.
```bash
waveform-cli open_waveform foo.vcd --alias w \
  -- read_signal w TOP.data_bus --time-indices 10,20,30,40 \
  -- read_signal w TOP.result   --time-indices 10,20,30,40
```
### 3. Find When a Specific Condition Occurs
Use `find_conditional_events` with an arbitrary boolean expression to locate the exact time indices at which a condition becomes true.
```bash
waveform-cli open_waveform foo.vcd --alias w \
  -- find_conditional_events w "TOP.valid == 1'b1 && TOP.ready == 1'b1" --limit 50
```
### 4. Detect Rising and Falling Edges
Use `\$past()` inside a condition to compare the current value against the previous time step, enabling edge detection.
```bash
# Rising edge
waveform-cli open_waveform foo.vcd --alias w \
  -- find_conditional_events w "TOP.clk == 1'b1 && \$past(TOP.clk) == 1'b0" --limit 100
# Falling edge
waveform-cli open_waveform foo.vcd --alias w \
  -- find_conditional_events w "TOP.clk == 1'b0 && \$past(TOP.clk) == 1'b1" --limit 100
```
Additional conditions can be appended with `&&` to restrict results to edges where other signals also satisfy a predicate — for example, sampling data only on rising clock edges where a valid signal is asserted.
---
## Example: Finding Valid Input Cycles and Reading Data
Goal: determine on which rising clock edges `input_valid` is high, then read the corresponding input data.
**Step 1** — Find all rising edges of `clk` where `input_valid` is asserted:
```bash
waveform-cli open_waveform sim/dut.vcd --alias w \
  -- find_conditional_events w \
     "TOP.clk == 1'b1 && \$past(TOP.clk) == 1'b0 && TOP.input_valid == 1'b1" \
     --limit 100
```
This yields a list of time indices, e.g. `42, 44, 46, 48`.
**Step 2** — Read the data bus at those indices:
```bash
waveform-cli open_waveform sim/dut.vcd --alias w \
  -- read_signal w TOP.input_data --time-indices 42,44,46,48
```
