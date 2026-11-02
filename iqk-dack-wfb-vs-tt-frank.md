# RTL8812EU bring-up: IQK, DACK, and how wfb-ng's register writes differ from tt-frank's

Written for engineers working on the tt-frank air unit (Luckfox Pico Mini B + RTL8812EU).
Status: 2026-11-03. Everything below was measured on one board and one module; where a
statement is an inference rather than a measurement it is marked **(inference)**.

"wfb-ng" here means the stack the board ran before tt-frank: the vendor kernel driver
`8812eu` (source: `rtl88x2eu`) plus `wfb_tx` injecting frames through its monitor-mode
interface. `wfb_tx` itself writes no registers; every register write comes from the driver.
"tt-frank" means tt-frank on top of devourer, a userspace re-implementation of the driver
over libusb.

## Summary

1. **IQK completes at 10 and 5 MHz but reports failure flags every time when it is run
   in-band** (8 of 8 completed runs at 10 MHz had a non-zero `fail_step`). Calibrating at
   20 MHz first and narrowing afterwards halves that (16 of 32 clean). The failure flags do
   **not** predict whether TX works. No in-band measurement exists for 5 MHz.
2. **The vendor driver never runs IQK in monitor mode.** There is no IQK traffic in its
   probe, in `ip link up`, or in three channel changes. devourer runs IQK on every start.
3. **DACK, however, is run by both, with the same registers.** The vendor does it once at
   driver load (`halrf_init`), devourer once at bring-up. They match write for write.
4. The register-write differences that mattered were in the *transport and ordering*, not
   in which registers are written: a dummy write to `0x04E0` after every always-on-register
   access, serialised transfers during power-up and the efuse read, one efuse read after
   firmware download, and calibrating at 20 MHz.
5. Reliability on the test board: original code 1 start in 6 worked; with the fixes
   27 of 36 with calibration on, 22 of 24 with DACK and IQK skipped. The same build also
   varied between sessions (10/12 to 1/12), which is unexplained.

## 1. Will IQK fail at 10 and 5 MHz?

### What IQK is, and why width matters

IQK calibrates I/Q imbalance and local-oscillator leakage of the TX and RX chains. devourer's
`Halrf8822e::phy_iq_calibrate` is a register-for-register port of the vendor's
`halrf_iqk_8822e.c`: three independent reviews of the port found the command words, the
done/fail polling, every `fail_step` bit and the retry loop identical. `fail_step` bits:
0 = LOK1 failed, 1 = TX-K failed, 2 = RX gain search failed, 3 = RX-K failed (OR-ed over both
paths).

The IQK command words are chosen from the **bandwidth field of RF register 0x18**. For 5 and
10 MHz that field still reads "20 MHz", because the RF is always tuned as a 20 MHz channel and
only the baseband clocks are divided. So at narrowband IQK sends the 20 MHz command set while
the DAC, ADC and filter already run at half or quarter rate **(inference)** — a combination
the vendor driver never exercises.

### Measurements

| IQK ran at | starts | IQKs that completed | `fail_step == 0` | other |
|---|---|---|---|---|
| 10 MHz in-band (boot at the profile width) | 12 | 8 | **0 of 8** (`0x03` ×6, `0x02`, `0x04`) | 3 starts hung in the calibration stage, 1 took 10 s |
| 20 MHz, then narrowed to 10 / 5 MHz | 36 | 32 | **16 of 32** | 1 hang, 1 slow DACK |

- IQK never hung or threw in either flow; failures show up as flags.
- At 5 MHz there is no in-band run at all; all 5 MHz data are "calibrate at 20 MHz, then narrow"
  (6 of 11 clean).
- `fail_step` did not predict TX: runs with `0x0b`, `0x0e`, `0x0f` transmitted normally, and
  starts with `0x00` went dead (for example a 5 MHz start with `0x00` sent 202 frames and then
  failed every transfer).
- **Not measured:** whether the 20 MHz coefficients are as good on air at 5/10 MHz as
  coefficients computed in-band would be, or whether skipping IQK costs EVM or range. That
  needs a ground-station comparison.

### Recommendation

Run IQK at 20 MHz and narrow afterwards (the vendor's tune → calibrate → dividers → BB reset
order, also documented in devourer `docs/narrowband.md`). tt-frank's stable change does this in
`Radio::init`. Do not calibrate in-band at 5 or 10 MHz.

## 2. wfb-ng's register writes versus tt-frank's

Measured by capturing the vendor driver's traffic with usbmon (`CONFIG_USB_MON` kernel, binary
interface so bulk payloads are complete) during `insmod`, `ip link up`, monitor mode and channel
changes, and comparing it with devourer's.

| | wfb-ng (vendor driver) | tt-frank before | tt-frank stable |
|---|---|---|---|
| Who issues the writes | kernel driver | devourer, libusb | devourer, libusb |
| Transfer style | one synchronous control transfer at a time | pipelined batch, several in flight | pipelined, except power-up and the efuse window |
| `0x04E0` dummy write after each access to `0x00–0xFF` / `0x1000–0x10FF` | **yes** (1,267 for 1,270 accesses) | none | yes |
| Power-on | reads chip state, wakes resident firmware, enables from the parked state, about 1.2 ms | card-disable always, then enable | card-disable skipped when already parked, no added pauses |
| Firmware download | twice per load (a first pass to read a hidden report, power-off, then the real one); 4,144-byte chunks with a minimal TX descriptor | once; descriptor also set USE_RATE, DISDATAFB, LS | once; vendor descriptor |
| Efuse read | once, about 1.1 s after power-on, after firmware and MAC init | 26 ms after power-on, before firmware; a second window after the tables | once, after firmware and MAC init |
| BB / AGC / RF tables | handed to the firmware: 24 `CFG_PARAMETER` H2C commands and 19 bulk packets of 4,092 B; 68 distinct BB/RFK registers written by control transfer | about 5,000 individual control writes (370 distinct BB/RFK registers) | unchanged |
| DACK | host-driven, once at load | same | same, at 20 MHz |
| IQK | **none** | host-driven at every bring-up (about 1.2–1.5 s, 8,300 transfers) | same, at 20 MHz |
| Bandwidth | 20 MHz (`iw`: width 20 MHz) | 5/10 MHz from boot, via the baseband dividers | boots at 20 MHz, dividers applied after IQK |
| Size of the load | about 7,100 control transfers plus 69 bulk (2.4 s paced) | about 13,900 control transfers | about 19,750 including the `0x04E0` writes |

What the tests showed about these differences:

- Replaying the vendor's exact captured transfers with plain libusb (no devourer, no kernel
  driver) through power-on, firmware, tables and calibration succeeded 50 of 50. The chip, the
  supply, the hub and libusb are fine; the failures came from what devourer sent.
- Replaying devourer's own recorded table writes inside the vendor flow also worked (10 of
  10), so the table method is not the problem.
- Turning only the `0x04E0` write off in the stable build brought the `iostream error` back
  at the second start. Removing the double efuse window and the early efuse read removed the
  deaths at the efuse registers (`0x0000`, `0x00A4`).
- Booting devourer from the vendor's recorded boot sequence initialised cleanly 24 of 24
  times, but TX then stalled after exactly 202 frames: the vendor's MAC TX queue and page
  layout (registers `0x0204–0x020C`, `0x0420–0x0456`, `REG_CR` network type) is not the one
  devourer's frame path was built for.

## 3. DACK

### What it is

DACK is the DAC/ADC calibration of the analog front end (`halrf_dac_cal_8822e`). It runs per
RF path (A and B) in three steps:

1. **ADDCK** — ADC DC-offset calibration: the ADC input is shorted, the DC level is read and
   a compensation word is written back.
2. **MSBK** — DAC MSB (coarse level) calibration.
3. **DADCK** — DAC DC-offset calibration.

MSBK and DADCK run with the baseband clock temporarily forced to 160 MHz (`0x9B4` field set to
`0xDB`) and restored afterwards; each waits for a hardware done flag and the results are backed
up and reloaded. It is independent of IQK, uses different registers (the AFE bank `0x3804…`,
`0x3830…`, `0x380C`, `0x3904…`, done flags `0x385C/0x388C/0x3870/0x38A0`), and runs before IQK.

### How each side handles it

| | wfb-ng | tt-frank / devourer |
|---|---|---|
| Where | `halrf_init` → `halrf_dack_trigger(force=false)` → `halrf_dac_cal_8822e`, at driver load, before any channel is set. The copy in the connect trigger is commented out, so it is not repeated. | `Halrf8822e::dac_calibrate`, called from `rtw_hal_init` after the RF cal-init block and before the channel is set. |
| Register activity in the captures | 2 DACK-clock writes, writes to `0x3804` ×6, `0x3830` ×2, `0x380C` ×4, `0x3904` ×6, 42 polls of the done flags | 2 DACK-clock writes, `0x3804` ×7, `0x3830` ×3, `0x380C` ×4, `0x3904` ×7, 37 polls |
| Result | one pass | verify (`dack_checkfail`) and up to 10 retries; logs `DACK complete (retries=…, msbk_to, dadck_to, addck_to)` |
| Cost when it works | not measured separately | 90–120 ms, about 690 transfers |

So DACK is the same operation on both sides; unlike IQK there is no behavioural difference to
port.

### What goes wrong

- devourer's poll loops give up after 10,000 iterations, and each iteration is a USB read plus
  a sleep, about 1 ms. A done flag that never sets therefore costs about 10 s per step. That
  matches DACK stages of 10.0–15.8 s seen in the logs **(inference)**; a retrying DACK can stay
  in the stage far longer, which is how several starts looked "hung" at 26 s.
- Frequency: 3 hangs and 1 slow DACK in 12 starts when calibrating in-band at 10 MHz; 1 hang
  and about 3 slow ones in 36 starts when calibrating at 20 MHz first. A working DACK took
  93 ms in the stable run.
- Skipping DACK on its own did not help (the one start with DACK skipped and IQK on went
  dead and then silent), and skipping IQK alone gave 6 of 8 with DACK still running. Neither
  calibration alone explains the instability.

## 4. What is still open

- Why a DACK done flag sometimes never sets on a start devourer otherwise brought up cleanly.
- Why the same build gave 10/12, 8/12 and 1/12 in different sessions.
- Whether IQK at 20 MHz and skipping IQK entirely are acceptable on air at 5/10 MHz.
- The firmware checksum failure (about 1 start in 8) and dead TX after a completed init
  (about 1 in 8 with calibration on).
- Which of the individual boot fixes beyond the `0x04E0` write are strictly needed: only the
  `0x04E0` write was ablated alone.

## Correction to earlier notes

Earlier write-ups from this investigation (`REPORT.md` and `README.md` under
`exp/usb-replay/stable-init`, the comment on `skip_dack` in devourer's `DeviceConfig.h` and in
tt-frank's `Config.h`, the `skip_dack` row in `docs/CONFIG.md`) say that "the vendor driver runs
no DACK/IQK in monitor mode". **The IQK half is right; the DACK half is wrong.** That claim came
from looking for writes to `0x1BD4/0x1BD8`, which are IQK's CFIR index port, not DACK registers.
The captures above show the vendor runs DACK with the same registers as devourer. Skipping
DACK therefore removes a step the vendor performs; the measured rates with both calibrations
skipped (22 of 24) still stand, but they do not show that DACK is the cause.

## Method and files

- Vendor driver source: `third_party/devourer/reference/rtl88x2eu`.
- Captures: `exp/usb-replay/usbmon_cap.c` (binary usbmon, full bulk payloads) and
  `convert_capture.py`; replay with `usb_replay2`; register dump with `usb_peek`.
- devourer: `src/jaguar3/Halrf8822e.cpp` (`phy_iq_calibrate`, `dac_calibrate`),
  `src/jaguar3/HalJaguar3.cpp` (`rtw_hal_init`, `run_iqk`).
- All numbers are start counts from 12-start loops on one board, 20 s of transmit per start;
  they are small samples.
