# Nios V Runtime HVIO IOPLL Reconfiguration: 100 MHz to 50 MHz

This project demonstrates runtime reconfiguration of an Agilex 5 HVIO IOPLL
from a **100 MHz output to a 50 MHz output** under Nios V control. It is also a
reference design for oscilloscope-based transient analysis of the interval in
which the output divider changes, the PLL may lose lock, and the output settles.

> Important frequency distinction: the board input `CLOCK0_50` is **50 MHz**.
> The IOPLL initially multiplies that reference to a 3.2 GHz VCO and divides it
> to produce a **100 MHz C0 output**. “100 MHz input” in older notes refers to
> the initial observed PLL output, not the physical reference-clock input.

The separate, non-destructive 50 MHz to 10 MHz derivative is available at
[iopll_niosv_50_to_10_transient](https://github.com/Jiayhocoding/iopll_niosv_50_to_10_transient).

## System architecture

```text
CLOCK0_50 (50 MHz)
        |
        +--> Platform Designer system --> Nios V
        |                              --> nios_iopll_bridge
        |                                      |
        |                               command handshake
        |                                      |
        +--> target_iopll <-------------- hvio_master
                 |                            |
                 +--> C0 output              +--> HVIO serial Avalon-MM pins
                 +--> locked

Scope outputs: D[0] = C0, D[1] = locked, D[2] = firmware marker
```

The Nios V processor does not directly drive the unusual fixed-cycle HVIO
interface. Software writes a small memory-mapped bridge. The bridge transfers a
command into `hvio_master`, which generates the five-cycle preamble, address,
four data bytes, mandatory hold, and idle cycles required by the IOPLL.

Relevant sources:

- `golden_top.v`: top-level IOPLL, bridge, HVIO master, GPIO routing;
- `rtl/nios_iopll_bridge.sv`: Nios V Avalon-MM command/status bridge;
- `rtl/hvio_master.sv`: low-level fixed-cycle HVIO transaction engine;
- `ip/target_iopll/target_iopll.ip`: static 100 MHz IOPLL configuration;
- `software/nios_iopll_external_rst_diagnostic/main.c`: verified low-level
  reference implementation; and
- `software/nios_iopll_signaltap_capture/main.c`: historical experimental
  capture variant whose address convention must not be copied blindly.

## PLL configuration

The generated target IOPLL uses:

| Item | Initial state | Reconfigured state |
|---|---:|---:|
| Physical reference | 50 MHz | 50 MHz |
| M counter | 64 | unchanged |
| N counter | 1 | unchanged |
| VCO | 3.2 GHz | unchanged |
| C0 total divide | 32 | 64 |
| C0 high count | 16 | 32 |
| C0 low count | 16 | 32 |
| C0 output | 100 MHz | 50 MHz |
| Nominal period | 10 ns | 20 ns |

Only C0 changes. Keeping M, N, and the VCO unchanged isolates the output-divider
transition and avoids changing the VCO operating point.

The C0 register is read-modify-written. Its high and low count fields are
replaced while unrelated bits are preserved. In the verified implementation,
the target count is high=32 and low=32 for divide-by-64.

## HVIO PLL Register Addressing

This section is intentionally explicit because the repository contains
historical firmware using two different numeric forms for the same registers.

There are three address domains:

1. **Documented/register byte offset** — the offsets printed in the Agilex 5
   Clocking and PLL User Guide, such as C0 at `0x5C`.
2. **Low-level Avalon-MM word address** — the 9-bit address consumed by the
   verified `hvio_master`, such as C0 at `0x17`.
3. **Nios V software/MMIO address** — the CPU address of a register inside
   `nios_iopll_bridge`, for example bridge base plus its ADDRESS-register
   offset. This selects a bridge register; it is not an HVIO PLL register
   offset.

For the proven low-level datapath, convert exactly once:

```text
word_address = documented_byte_offset >> 2
```

| Documented byte offset | Low-level word address | Register |
|---:|---:|---|
| `0x10` | `0x04` | Enable read/write |
| `0x48` | `0x12` | Enable recalibration |
| `0x54` | `0x15` | Clock gating |
| `0x58` | `0x16` | Clear status |
| `0x5C` | `0x17` | C0 |
| `0x60` | `0x18` | C1 |
| `0x80` | `0x20` | PLL reset |
| `0x88` | `0x22` | Recalibration request |

### Where conversion occurs

Inspection of the actual RTL shows that neither hardware block scales the
address:

- `nios_iopll_bridge.sv` stores the value written to its ADDRESS register and
  transfers that same 9-bit value as `cmd_address`.
- `hvio_master.sv` captures `cmd_address` and drives it unchanged on the HVIO
  core Avalon address pins.

Therefore, when firmware constants are documented byte offsets, software must
shift right by two before writing the bridge ADDRESS register. If firmware
constants are already low-level word addresses, it must not shift again.
Conversion must happen **exactly once**.

The generated BSP places the bridge at CPU MMIO base `0x90040`. Its local
register offsets are:

| Bridge offset | Purpose |
|---:|---|
| `0x00` | CONTROL |
| `0x04` | HVIO word ADDRESS |
| `0x08` | write data |
| `0x0C` | read data |
| `0x10` | STATUS |
| `0x14` | diagnostic/external-reset control |

Thus a CPU write to `0x90044` loads the bridge ADDRESS register. The data placed
there should be `0x17` to access PLL C0; `0x90044` is not itself the C0 address.

Historical `main.c` variants are experiments, not authoritative descriptions
of the active datapath. One passes `0x5C` directly and another uses `0x17`.
Follow the bridge/master RTL and the successfully tested word-address variant.

Incorrect scaling can still produce an apparently completed bus transaction
while selecting the wrong PLL register. The consequences include misleading
readback, incorrect divider programming, PLL lock loss, and failure to recover
during recalibration.

## Bridge registers and bit fields

### CONTROL (`base + 0x00`)

| Bit | Meaning |
|---:|---|
| 0 | start command |
| 1 | write when 1, read when 0 |
| 8 | clear done |
| 9 | clear error |
| 10 | clear rejected-command flag |

### STATUS (`base + 0x10`)

| Bit | Meaning |
|---:|---|
| 0 | busy |
| 1 | done |
| 2 | transaction error |
| 3 | synchronized PLL lock |
| 4 | command rejected because another command was active |

The status lock bit passes through synchronizers and software polling. Use the
direct `D[1]` output for oscilloscope timing rather than timestamping status-bit
changes in firmware.

## Verified Nios V reconfiguration sequence

The proven diagnostic firmware follows this sequence:

1. Confirm that the bridge is idle and clear sticky completion/error flags.
2. Enable HVIO register read/write access at word address `0x04`.
3. Enable recalibration at word address `0x12`.
4. Read C0 at word address `0x17`.
5. Preserve unrelated C0 bits and program high=32, low=32.
6. Assert PLL reset through word address `0x20` for at least the documented
   minimum; the diagnostic implementation holds it for 1 microsecond.
7. Clear calibration/status state through word address `0x16`.
8. Request recalibration through word address `0x22`.
9. Release reset, observe lock deassertion if it occurs, and wait for lock to
   return with a timeout.
10. Read C0 back and verify the programmed count fields.
11. Clear/disable temporary access and recalibration controls as required.

The exact ordering in the firmware should remain the executable reference. Any
new experiment should check every bridge transaction for timeout, error, or
rejection rather than assuming completion means the intended PLL register was
accessed.

## Transaction timing

At the 50 MHz controller clock, one cycle is 20 ns. A low-level write contains:

- five preamble cycles;
- four LSB-first data-byte cycles;
- one additional D3 hold cycle; and
- five idle cycles.

That is 15 controller cycles, approximately 300 ns, before software and bridge
handshake overhead. This is protocol latency, not PLL relock time. Reset,
recalibration, and analog lock acquisition occur on longer and device-dependent
timescales and must be measured.

## Oscilloscope connections

Use three channels with a common ground:

| Scope channel | FPGA signal | Top-level pin | Purpose |
|---|---|---|---|
| CH1 | IOPLL C0 output | `D[0]`, `PIN_BK31` | Observe 10 ns becoming 20 ns |
| CH2 | direct IOPLL `locked` | `D[1]`, `PIN_BE43` | Measure lock loss and reacquisition |
| CH3 | firmware trigger marker | `D[2]`, `PIN_BF29` | Mark the start of reconfiguration |

The outputs are assigned 3.3 V LVCMOS. Verify the board connector and voltage
before probing. Use short ground springs, high-bandwidth probes, and 10x mode.
Trigger on the CH3 rising edge and capture enough pre-trigger history to show
several stable 100 MHz cycles.

Measure these timestamps from a single acquisition:

- **T0:** marker rising edge;
- **T1:** first C0 period that differs materially from 10 ns;
- **T2:** direct lock falling edge, if present;
- **T3:** direct lock rising edge;
- **T4:** first point after which C0 remains within the chosen 20 ns tolerance.

Then report:

```text
command-to-output response = T1 - T0
lock-low duration          = T3 - T2
total settling time        = T4 - T0
```

Do not classify every stretched period during reset as an unintended glitch.
Separate the expected output interruption while reset/recalibration is active
from runt pulses, double edges, non-monotonic edge spacing, or cycles occurring
after lock returns but before the output becomes stable.

## Expected waveform

Before the command, CH1 should show approximately 10 ns periods and CH2 should
be high. CH3 then rises. During reset/recalibration, C0 may pause or produce
irregular edge spacing, and `locked` may fall. After reacquisition, `locked`
should rise and C0 should settle to approximately 20 ns periods.

The RTL and documentation cannot supply a defensible numerical lock or settling
time. Those values depend on the programmed sequence, device, PVT conditions,
board signal integrity, and measurement threshold. Record minimum, maximum, and
typical values over many transitions rather than relying on a single capture.

## 50 MHz to 10 MHz derivative

The derivative project changes the static C0 state to divide-by-64 (50 MHz) and
the runtime C0 state to divide-by-320 (10 MHz), using high=160 and low=160. The
VCO remains at 3.2 GHz. Its 20 ns to 100 ns period change is substantially easier
to distinguish on a scope.

It also adds a dedicated firmware application that performs the byte-offset to
word-address conversion exactly once, a transaction/mapping testbench, and a
README tailored to that experiment. Keeping it in a separate repository
preserves this original working implementation.

## References

- [Altera Agilex 5 Clocking and PLL User Guide: HVIO read/write timing](https://docs.altera.com/r/docs/813671/25.1.1/clocking-and-pll-user-guide-agilextm-5-fpgas-and-socs/read-and-write-operations-via-avalon-memory-mapped-interface)
- [Enabling HVIO IOPLL register access](https://docs.altera.com/r/docs/813671/25.1.1/clocking-and-pll-user-guide-agilextm-5-fpgas-and-socs/enabling-reconfiguration-for-the-desired-i/o-pll)
- [Reconfiguring and resetting the IOPLL](https://docs.altera.com/r/docs/813671/25.1.1/clocking-and-pll-user-guide-agilextm-5-fpgas-and-socs/reconfiguring-the-i/o-pll)
- [Counter address and bit-field table](https://docs.altera.com/r/docs/813671/25.1.1/clocking-and-pll-user-guide-agilextm-5-fpgas-and-socs/divide-settings-and-the-corresponding-data-bit-setting-for-reconfiguration)
