# Full-Duplex UART with FIFO Buffering & Packetized Error Handling

A synthesizable, fully synchronous **UART (Universal Asynchronous Receiver/Transmitter)** core written in Verilog HDL, featuring independent transmit/receive datapaths, a 16x-oversampling baud rate generator, 16-deep circular-buffer FIFOs for clock-domain-safe data handoff, and a packetized RX FIFO that staples parity-error metadata directly onto received bytes for interrupt-free fault isolation.

---

## Architecture Overview

```
                        ┌────────────────────┐
                        │     baud_clock      │
                        │  (16x oversampling  │
                        │   tick generator)    │
                        └─────────┬───┬────────┘
                             tx_e │   │ rx_e
                  ┌────────────────┘   └────────────────┐
                  ▼                                      ▼
     ┌────────────────────┐                 ┌────────────────────┐
     │    transmitter      │                 │      receiver       │
     │  (8-E-1 TX FSM,     │                 │  (8-E-1 RX FSM,     │
     │   2 stop-bit IFG)   │                 │   parity check)     │
     └───────┬─────────────┘                 └──────────┬──────────┘
             │ rd_en_tx                        wr_en_rx  │
             ▼                                            ▼
   ┌───────────────────┐                        ┌───────────────────┐
   │  rx_fifo (WIDTH=8) │                        │ rx_fifo (WIDTH=9) │
   │   TX staging FIFO  │                        │  RX packet FIFO   │
   │  (data_in → tx)     │                        │ [parity | data]   │
   └─────────▲──────────┘                        └──────────┬────────┘
             │ w_e, data_in                                  │ r_e
             │                                                ▼
   ┌─────────┴──────────────────────────────────────────────────────┐
   │                          main_module (top)                       │
   │        clk, reset_n, w_e, r_e, rdy_clr, data_in[7:0] → d_out[8:0]│
   └───────────────────────────────────────────────────────────────┘
```

## Key Features

- **Independent Tx/Rx channels** running off a single shared, custom 16x-oversampling baud rate generator — no external clock divider IP required.
- **16-deep circular-buffer FIFOs** on both the transmit-staging and receive-packet paths, decoupling the CPU/host-facing interface from the serial bit clock and enabling safe cross-domain data transfer.
- **8-E-1 framing** (1 start bit, 8 data bits, 1 even-parity bit) implemented as an explicit Mealy/Moore-style FSM (`idle → start → data → par → stop1 → stop2`) with a **2-stop-bit inter-frame gap** to guard against receiver timing drift during continuous back-to-back transmission.
- **Packetized 9-bit RX FIFO** — the receiver appends a computed parity-error bit to every received byte before it's pushed into the FIFO, so a downstream reader can identify and isolate a corrupted byte *at read time* without needing a separate interrupt or error-flag register.
- **Fully synchronous, single-clock design** — every module (`baud_clock`, `transmitter`, `receiver`, `rx_fifo`) is clocked on `posedge clk`, simplifying timing closure and static timing analysis.
- **Parameterized FIFO** — a single `rx_fifo` module (`parameter WIDTH`) is reused for both the 8-bit TX staging queue and the 9-bit RX packet queue, avoiding code duplication.

## Module Breakdown

| Module | File | Responsibility |
|---|---|---|
| `main_module` | `main_module.v` | Top-level integration: wires the baud generator, transmitter, receiver, and both FIFOs into a single host-facing UART peripheral. |
| `baud_clock` | `baud_clock.v` | Free-running counter that derives an RX sample tick (`rx_e`, 1-in-326 clocks) and, by further dividing that by 16, a TX bit tick (`tx_e`) — implementing classic 16x oversampling for reliable mid-bit sampling on receive. |
| `transmitter` | `transmitter.v` | 6-state FSM (`idle, start, data, par, stop1, stop2`) that pulls a byte from the TX FIFO, serializes it LSB-first, appends an even-parity bit, and frames it with a start bit and **two** stop bits. Exposes a `busy` flag for host-side flow control. |
| `receiver` | `receiver.v`* | Samples the serial line at the oversampled rate, reconstructs the byte + parity, and pushes a 9-bit `{parity_error, data}` word into the RX FIFO along with a `rdy` strobe (cleared via `ready_clr`). |
| `rx_fifo` | `rx_fifo.v` | Parameterized (`WIDTH`) circular-buffer FIFO with `front`/`rear` pointers, a 5-bit occupancy counter, and combinational `full`/`empty` flags. Instantiated twice — once as an 8-bit TX queue, once as a 9-bit RX packet queue. |

\* *`receiver.v` is instantiated by `main_module` and referenced throughout this README to describe the complete datapath; include it in this directory alongside the other four files when building/simulating the project.*

## Interface (`main_module`)

| Signal | Dir | Width | Description |
|---|---|---|---|
| `clk` | in | 1 | System clock |
| `reset_n` | in | 1 | Active-low synchronous reset |
| `w_e` | in | 1 | Write-enable: push a byte into the TX FIFO |
| `data_in` | in | 8 | Byte to transmit |
| `r_e` | in | 1 | Read-enable: pop a word from the RX FIFO |
| `rdy_clr` | in | 1 | Clears the receiver's `rdy` flag |
| `full_tx_fifo` | out | 1 | TX FIFO full — host should stall writes |
| `empty_rx_fifo` | out | 1 | RX FIFO empty — nothing to read |
| `busy` | out | 1 | Transmitter is actively shifting a frame |
| `rdy` | out | 1 | A new received byte is available |
| `d_out` | out | 9 | `{parity_error, rx_data[7:0]}` — packetized received word |

## Design Notes / Trade-offs

- **Baud generation is fixed at design time** (`counter_rx == 9'd325`, `tick_counter == 4'd15`) — divide values are hard-coded rather than parameterized. Making `BAUD_DIV` a module parameter would be a natural next step to support multiple baud rates from one core.
- **FIFO depth is fixed at 16** entries via `mem[15:0]`; `WIDTH` is parameterized but depth is not — a good candidate for a `DEPTH` parameter in a future revision.
- **Error handling is non-blocking**: rather than stalling or raising a CPU interrupt on a parity fault, the error bit rides along with the data in the FIFO, letting the consumer decide per-byte whether to discard, retry, or log — useful in streaming/burst scenarios where halting the pipe is costly.

## Verification

The design was exercised in simulation against:
- FIFO boundary conditions (`full`/`empty` flag correctness across simultaneous read+write, back-to-back pushes/pops)
- Continuous back-to-back frame transmission to validate the 2-stop-bit inter-frame gap prevents bit-slip on the receive side
- Injected parity errors to confirm the 9-bit RX FIFO correctly tags corrupted bytes without stalling the pipeline

