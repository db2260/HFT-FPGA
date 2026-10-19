# HFT on FPGA — Basys 3 (Artix-7)

UART-driven order-book + trading-logic pipeline on an FPGA.
A Python GUI plays the "exchange". The FPGA replies with trade decisions.

---

## 1. How it works

```
PC (src/python/main.py)  --UART 115200 8N1-->  uart_rx -> parser_top -> order_book_wrapper
                                                                           |
PC (decoded + plotted)   <--UART 115200 8N1--  uart_tx <- custom_msg_gen <- Top_Trading
```

| Stage | File | Job |
|---|---|---|
| UART RX/TX | `uart_rx.v`, `uart_tx.v` | 100 MHz clk, 115200 baud, 8N1 |
| Parser | `parser_top.v` | Decodes ITCH-style ADD / CANCEL / EXECUTE |
| Order book | `order_book_wrapper.v`, `order_book.v`, `add_order.v`, `decrease_order.v`, `memory_manager.v` | 4 books, BRAM per book, best price per stock |
| Trading logic | `D_Wrap.sv` (`Top_Trading`), `D_Top.sv` (`multi_strategy_trading`) | Picks quantity + buy/sell per stock |
| Reply | `costom_msg_generator.v` | 4 × 7-byte messages per update |
| Top | `HFT_top.v` | Ports: `clk`, `rst`, `rx`, `tx` |

---

## 2. Repo layout (actual)

```
src/verilog/   RTL, constraints, IP (.xci), testbenches
src/python/    Exchange simulator GUI + helpers
docs/          Final report + slides (PDF)
```

---

## 3. FPGA build

### Prerequisites
- Vivado (any version that supports Artix-7 + ILA + Block Memory Generator)
- Basys 3 board
- Micro-USB cable (JTAG + UART share it)

### Step 1 — Clone
```
cd HFT-FPGA
```

### Step 2 — Create a new project ⚠
Part: **`xc7a35tcpg236-1`** (Basys 3).

> The shipped `.xci` files target `xc7a100t-csg324-1`. See step 5.

### Step 3 — Add sources
Add from `src/verilog/`:

| Add as | Files |
|---|---|
| Design sources | `HFT_top.v`, `uart_rx.v`, `uart_tx.v`, `parser_top.v`, `order_book_wrapper.v`, `order_book.v`, `add_order.v`, `decrease_order.v`, `memory_manager.v`, `costom_msg_generator.v`, `D_Wrap.sv`, `D_Top.sv`, `constants.v` |
| IP | `blk_mem_gen_0.xci`, `ila_0.xci` |
| Constraints | `Constriant_File.xdc` (after step 6) |
| Simulation only | `parser_top_tb.v`, `uart_top_tb.v` |

Set top module: `HFT_top`.

Do **not** add `HFT_top.dcp` (18 MB prebuilt checkpoint). It is not needed.

### Step 4 — Fix the `include` path ⚠
Six files contain:
```verilog
`include "../../../new/constants.v"
```
That path only existed in the original authors' Vivado tree. Verilator cannot resolve it from this repo layout.

Fix (run in `src/verilog/`):
```
# Linux / macOS / Git Bash
sed -i 's#"../../../new/constants.v"#"constants.v"#' add_order.v decrease_order.v memory_manager.v order_book.v order_book_wrapper.v parser_top.v
```
Then in Vivado: right-click `constants.v` → **Set Global Include**.

### Step 5 — Re-target the IP ⚠
Both `.xci` files are generated for `xc7a100t-csg324-1`.

After setting the Basys 3 part:
1. IP Sources → select each IP → **Reset Output Products**
2. **Generate Output Products**
3. Accept any "upgrade IP" prompt

| IP | Settings in the `.xci` |
|---|---|
| `blk_mem_gen_0` | Single-port RAM, 32-bit wide, depth 255 |
| `ila_0` | 11 probes, depth 16384 |

No ILA needed? Delete the `ila_0 your_instance_name (...)` block at the bottom of `HFT_top.v`, and remove `ila_0.xci`.

### Step 6 — Fix pin constraints ⚠
`Constriant_File.xdc` uses pins `E3 / N17 / C4 / D4`.
Those match the Nexys A7 / Nexys 4 DDR, **not** Basys 3.

Basys 3 pins (from Digilent's master XDC — confirm against your board manual):

| Port | Basys 3 pin | Board function | Shipped pin |
|---|---|---|---|
| `clk` | `W5` | 100 MHz osc | `E3` |
| `rst` | `U18` | BTNC (active high) | `N17` |
| `rx` | `B18` | USB-UART → FPGA | `C4` |
| `tx` | `A18` | FPGA → USB-UART | `D4` |

Keep the existing `create_clock -period 10.000` and `LVCMOS33` lines.

### Step 7 — Build and program
Synthesis → Implementation → Generate Bitstream → Hardware Manager → Program.

Press **BTNC** once after programming to reset.

---

## 4. Python simulator

### Install
```
pip install pyserial matplotlib
```
`tkinter` must also be present.
- Windows / macOS (python.org build): included
- Debian/Ubuntu: `sudo apt install python3-tk`

NumPy is **not** needed.

### Run
```
cd src/python
python main.py
```
Run from `src/python`. It imports sibling modules and reads `config.ini` from the current directory.

> ⚠ Do not run `exchange_simulator.py`. It is an older monolithic version and has a syntax error at line 679.

### Use
1. **Refresh** → pick the Basys 3 COM port → **Connect**
2. Click **Load CSV** (e.g. `book_data_rand3.csv`) or leave empty to auto-generate
3. **Start Simulation**
4. Watch tabs: *Sent Packets*, *Received Packets*, *Order Book*, *Real-time Plots*

### `config.ini`
| Key | Shipped | Meaning |
|---|---|---|
| `baud_rate` | 115200 | Must match RTL |
| `packet_delay` | 0.5 | Seconds between packets |
| `num_packets` | 1500 | Auto-generated packet count |
| `num_stocks` | 4 | Keep at 4 |
| `price_min` / `price_max` | 50 / 100 | Price range |
| `quantity_min` / `quantity_max` | 1 / 255 | 8-bit quantity |
| `cancel_probability` | 0.3 | Share of CANCEL packets |
| `buy_orders_only` | True | Generate only buys |
| `cancel_highest_price` | True | Always cancel the best order |

### CSV format
```
type,stock_id,order_id,is_buy,price,quantity
ADD,0,220,TRUE,30,140
CANCEL,...
```

---

## 5. Bring-up checklist

| # | Check | Why |
|---|---|---|
| 1 | Send ADDs for **all four stocks (0–3)** first | Trading logic fires only when every book has ≥ 1 order |
| 2 | Stock IDs are 0–3 only | 2-bit ID. Others are ignored |
| 3 | Order IDs 1–255 (simulator stays ≤ 200) | ID is used as BRAM address |
| 4 | Start with `packet_delay = 0.5` | Easy to see traffic. Lower it later |
| 5 | No reply? Use ILA probes: `rx`, `rx_ready`, `out_ready`, `best_price_valid`, `o_Valid`, `tx_start` | Shows where the pipeline stops |
| 6 | Reset is **active high** | BTNC pressed = reset |

---

## 6. Wire protocol

### PC → FPGA (little-endian)

**ADD — 37 bytes**
| Byte | Field | Value |
|---|---|---|
| 0 | Length | `0x25` (37) |
| 1 | Type | `0x82` |
| 2–3 | Stock ID | 0–3 |
| 4–5 | Tracking | random |
| 6–11 | Timestamp | ns since midnight |
| 12–19 | Order ID | 1–255 |
| 20 | Side | `0x41` buy, `0x42` sell |
| 21–24 | Shares | 1–255 |
| 25–32 | Symbol | `STOCKnn` padded |
| 33–36 | Price | `[frac, int, 0x06, 0x06]` |

**CANCEL — 24 bytes**
| Byte | Field | Value |
|---|---|---|
| 0 | Length | `0x18` (24) |
| 1 | Type | `0xA1` |
| 2–3 | Stock ID | 0–3 |
| 4–5 | Tracking | random |
| 6–11 | Timestamp | ns |
| 12–19 | Order ID | 1–255 |
| 20–23 | Cancelled shares | quantity |

The RTL also parses EXECUTE; the simulator does not send it.

### FPGA → PC
One update = **28 bytes** = 4 × 7-byte messages (one per stock).
Layout below is as decoded by the simulator (`packet_utils.py`):

| Byte | Field |
|---|---|
| 0 | Length (7) |
| 1 | Stock ID |
| 2 | 1 = buy, 0 = sell |
| 3–4 | Quantity (8.8 fixed-point) |
| 5–6 | Price (8.8 fixed-point) |

`packet_utils.parse_fpga_response_packet` decodes this.

---

## 7. Trading strategies (what the RTL actually has)

`HFT_top.v` hard-wires `.strategy_select(2'b00)`.
To try another, change that constant and rebuild.

| `strategy_select` | Strategy in `D_Top.sv` | Buy / sell |
|---|---|---|
| `2'b00` (default) | Equal weight | All **sell** |
| `2'b01` | Inverse-price weighted | All sell |
| `2'b10` | Price-weighted | All sell |
| `2'b11` | Volatility-weighted | Buy if price < moving avg, else sell |

Common settings:
- Capital: 1024 (8.8 fixed-point)
- Output price: best price + `0x0019` (≈ 0.1)
- Moving average length: 4 (set in `D_Wrap.sv`)

⚠ The PDF report describes something different: "Momentum (depth-8 moving average)" and a "Minimum Variance Portfolio". **MVP is not in this repo.** Per the report, only Equal-Weight and Momentum were validated on hardware; treat `2'b00` as the known-good path.

---

## 8. Known issues

| Issue | Detail |
|---|---|
| Include path | `../../../new/constants.v` — fix in §3 step 4 |
| Wrong board files | XDC + `.xci` are for a 100T device — §3 steps 5–6 |
| Divider | Report cites an AXI divider IP. The RTL uses `/` directly, with no divider IP. Expect timing pressure at 100 MHz |
| `tx_busy` | Used in `HFT_top.v` before it is declared (implicit net). Vivado warns. Add `wire tx_busy;` to silence |
| Moving-average length | `D_Top.sv` default is 3, but `D_Wrap.sv` overrides it to 4. 4 is what gets built |
| Throughput | Report states 400 orders/s. A 37-byte ADD takes ≈ 3.2 ms on a 115200 UART (10 bits/byte), so ≈ 310 ADD/s max. A 24-byte CANCEL allows ≈ 480/s |
| Latency | "≈ 3.2 µs" is FPGA core time (report: 100 MHz clock). UART wire time dominates end-to-end |
| Verilator | Reports errors in `parser_top.v` and `order_book.v` (mixed assignment styles). These are Verilator limits; Vivado accepts them |

---

## 9. Reported results (from the PDF — not re-measured)

| Metric | Value |
|---|---|
| Fmax | 104 MHz |
| LUTs | 5,440 (26 %) |
| LUTRAM | 366 (4 %) |
| FFs | 8,159 (20 %) |
| BRAM | 3 (6 %) |
| Power | 0.135 W |
| Core latency | ≈ 3.2 µs |

---

## 10. Simulation

`parser_top_tb.v` and `uart_top_tb.v` are self-contained testbenches.
Add as simulation sources, set the testbench as top, run behavioural simulation.
(They instantiate `parser_top.v`, so complete §3 step 4 first.)

---

## 11. Future work
- Ethernet instead of UART
- Buy + sell strategies beyond sell-only default
- NSE / BSE message formats
- Cancel / modify / more message types
- More than 4 stocks
- Full Minimum-Variance Portfolio on hardware
- Lower-latency order cancel (currently O(n) scan)

---

## 12. References
1. NASDAQ TotalView-ITCH 5.0 specification
2. Kahssay et al., "An HFT Accelerator" (2019)
3. Leber, Geib, Litz, "High frequency trading acceleration using FPGAs", FPL 2011
4. Xilinx, Artix-7 FPGAs Data Sheet (DS181)
5. Digilent, Basys 3 Reference Manual
6. `docs/HFT_FPGA_Final_Report.pdf`, `docs/HFT_Presentation.pdf`