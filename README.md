![gds](../../workflows/gds/badge.svg) ![docs](../../workflows/docs/badge.svg) ![test](../../workflows/test/badge.svg)

# Ternary MLP Inference Tile — Clock-Gated

The same 16-8-4 ternary-weight classifier as
[`tt-ternary-mlp`](https://github.com/allpcm/tt-ternary-mlp), with one change: the
320-bit weight register runs off a gated clock.

This tile exists to be compared against that one. Both are on the same Tiny Tapeout
shuttle — the ungated tile on TTSKY26d, this one on TTSKY26e, both on run CI-2612 —
so they land on the same die, from the same wafer, in the same package, built by the
same flow. One variable differs. That is the experiment.

**[Full datasheet and I/O protocol → `docs/info.md`](docs/info.md)**

---

## Why

Annotated power on the ungated tile measured **1.706 mW**, of which 82.5% is
internal power and 28.7% is clock. The weight register is 320 of the design's 419
flip-flops and contributes **0.0000%** of switching activity during inference — it
holds static data from the end of the load phase onward.

It was being clocked 50 million times a second anyway. A flip-flop burns internal
power on every clock edge whether or not its D input changes, and nothing in that
netlist gated a clock. Measuring real switching activity was expected to collapse
the power estimate and moved it 8.5%. Zero data toggles is not zero power.

## What changed

One register's clock, and nothing else. The logic, the pinout, the protocol, the
26-cycle latency and the trained network are all identical.

```verilog
wire wmem_en = w_shift | ~rst_n;

sky130_fd_sc_hd__dlclkp_1 u_wmem_icg (
    .CLK (clk), .GATE(wmem_en), .GCLK(wmem_clk)
);

always @(posedge wmem_clk) begin
    if (!rst_n) wmem <= {W_BITS{1'b0}};
    else        wmem <= {wmem[W_BITS-2:0], w_data};
end
```

Two details that are not arbitrary:

**Reset is in the enable term.** Gating on `w_shift` alone stops the clock while
`rst_n` is low, so the synchronous clear never sees an edge and the register leaves
reset holding whatever it held before. That failure would not show up in a test that
resets and then immediately loads weights — only on the first inference after a reset
that skipped a load.

**The `always` block has no enable term.** The gate carries it. That is what removes
the feedback mux from each of the 320 flops, and it is where the area saving comes
from.

## Verified results

From CI run [`45094551201`](https://github.com/allpcm/tt-ternary-mlp-cg/actions/runs/45094551201),
commit `59c8dbf` (LibreLane, sky130A):

| | |
|---|---|
| Tile size / clock | 2×2, 50 MHz |
| Latency | 26 cycles, fixed, input-independent |
| Synthesis | 1,525 cells, 19,068 µm² |
| Die area | 75,602 µm² (334.88 × 225.76 µm) |
| Flip-flops | 419 × `sky130_fd_sc_hd__dfxtp_2` |
| Clock gates | 1 × `sky130_fd_sc_hd__dlclkp_1` |
| Tiny Tapeout precheck | pass |
| Gate-level test | 200 vectors against the synthesized netlist, pass |
| Verilator `-Wall` | clean |

### Against the ungated tile

Synthesis-stage figures, same metric from the same tool on both designs:

| | ungated | gated | |
|---|---|---|---|
| cells | 1,834 | **1,525** | −309 |
| cell area | 22,700 µm² | **19,068 µm²** | −16.0% |
| flip-flops | 419 | 419 | — |
| die | 75,602 µm² | 75,602 µm² | — |

The flop count is unchanged, which is correct: clock gating does not remove
flip-flops. The 309 cells that disappeared are the enable multiplexers that used to
sit in front of the 320 weight flops. In the ungated netlist those flops carry an
enable; here they are plain `dfxtp_2` fed by a gated clock.

## What is NOT measured

**The power saving.** Flop-count proportion suggests gating recovers 0.6–0.9 mW of
the ungated tile's 1.7 mW. That is an estimate from cell counts, not a measurement,
and the one time an estimate was checked against a measurement on this design it was
wrong by 1.9×. The annotated-power run against this netlist has not been done yet.

**Anything on real silicon.** Chips are expected around 2027-05-25. Until then every
number here is from the flow, not from a part. The on-chip comparison this tile was
built for — supply current with each tile selected, taken differentially — also needs
a measurement method worked out: Tiny Tapeout tiles share a supply rail and the
multiplexer activates one design at a time, so per-tile power is not directly
instrumented.

## Build note worth keeping

The first version wrapped the clock-gating cell in a `pcm_clkgate` submodule. Yosys
was fine with it, synthesis completed, lint was clean — and then LibreLane died late
in place-and-route:

```
Connecting power net uio_oe to u_wmem_cg.u_icg/VPB… [INFO] Made 0 connections.
ERROR  add_global_connections failed to make any connections
       for 'u_wmem_cg.u_icg/VPB' to uio_oe.
```

`SetPowerConnections` tried to bind the cell's well-tie pin to a signal net. The
giveaway is the hierarchical instance path. Instantiating the cell flat in the top
module, with no wrapper and no power pins, fixed it. If you hand-instantiate a PDK
cell in a Tiny Tapeout project, do not put it inside a submodule.
