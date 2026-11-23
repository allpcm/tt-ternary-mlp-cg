## How it works

This is the clock-gated twin of the Ternary MLP Inference Tile. The logic is
identical — same weight-stationary ternary MLP, same 26-cycle fixed latency,
same pinout, same trained network. One thing differs: the 320-bit weight
register's clock is gated.

On the ungated tile, annotated power measured 1.706 mW, of which 82.5% is
internal power and 28.7% is clock. The weight register is 320 of the design's
419 flip-flops and contributes 0.0000% of switching activity during inference —
it holds static data from the end of the load phase onward. It was nonetheless
being clocked 50 million times a second, because a flip-flop burns internal
power on every clock edge whether or not its D input changes, and nothing in
that netlist gated a clock.

Here the register is driven by `sky130_fd_sc_hd__dlclkp_1`, an integrated clock
gating cell, enabled by `w_shift | ~rst_n`. Reset is in the enable term on
purpose: gating on `w_shift` alone would stop the clock while `rst_n` is low,
the synchronous clear would never see an edge, and the register would leave
reset holding whatever it held before.

## How to test

Identical to the ungated tile. Shift 320 weight bits in on `uio[1]` while
holding `uio[0]` high, load an activation byte on `ui[0..7]` with `uio[3]`
high, then strobe and read the class index. The cocotb bench runs the same 200
vectors at RTL and gate level.

The interesting test is not functional. Both tiles are on the same die, from
the same wafer, in the same package, built by the same flow — differing only in
this clock gate. Measure the supply current with each tile selected and the
difference is the cost of clocking a register that never changes.

## External hardware

None.
