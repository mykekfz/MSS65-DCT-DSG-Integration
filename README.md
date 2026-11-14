# MSS65 V452 + E90 M3 DCT (DME-only)

This patch lets an E60 M5 with the S85 and the stock MSS65 DME (V452) run an E90 M3 DCT. The car was SMG before. The TCU isn't flashed, the DCT is stock, and the calibration area of the DME isn't touched.

The DME stays coded as SMG. The patch sits in the program flash and takes the DCT's CAN messages and feeds them into paths the DME already has: the SMG status inputs and the DSC torque intervention. The DCT sees something close to an E9x M3 DME, and the rest of the E60 sees the same DME as before.

Made by **ZiMMie** for M5Board and Nam3Forum. Free to use, non-commercial only (see [Licence](#licence)).

**Value for Value:** it's free. If it saved you time or money, send back whatever it was worth to you.

[![Buy Me a Coffee](https://img.shields.io/badge/Buy%20me%20a%20coffee-Value%20for%20Value-FFDD00?style=for-the-badge&logo=buymeacoffee&logoColor=black)](https://buymeacoffee.com/e6xdonate)

---

## Contents

- [Status](#status)
- [What's in the repo](#whats-in-the-repo)
- [What you need](#what-you-need)
- [Acceleration sensor wiring](#acceleration-sensor-wiring)
- [How it works](#how-it-works)
- [CAN messages](#can-messages)
- [INJ side](#inj-side)
- [IGN side](#ign-side)
- [DSC and DCT faults](#dsc-and-dct-faults)
- [Launch control](#launch-control)
- [Cluster](#cluster)
- [XDF parameters](#xdf-parameters)
- [Stock V452 parameters with the DCT](#stock-v452-parameters-with-the-dct)
- [Memory map](#memory-map)
- [Flashing](#flashing)
- [Testing](#testing)
- [Logging](#logging)
- [How it went](#how-it-went)
- [What didn't work](#what-didnt-work)
- [Still to do](#still-to-do)
- [Shout-out](#shout-out)
- [Licence](#licence)

---

## Status

Build: **stage 3i**

| Function | Status |
|---|---|
| Pull away, clutch closed/open state | works |
| Upshift torque cuts | works |
| Downshift blips | works |
| DSC hill assist, no brake warning light | works |
| DCT fault CF21 (acceleration checksum) | fixed |
| DCT fault CF25 (handbrake timeout) | fixed |
| Launch control arms | works |
| Launch rpm limiter | tested in emulator, not on the road yet |
| M symbol shows P | works |
| Yellow transmission warning in P | gone with cluster coding |
| N shown in P | doesn't work, switched off ([see below](#what-didnt-work)) |
| Oil level measurement | should work now, not checked yet |

---

## What's in the repo

| File | |
|---|---|
| `V452_DCT_stage3i.bin` | Patched V452 full bin (5 MB), full flash |
| `V452_DCT_params.xdf` | TunerPro XDF for the patch settings, use it on the full bin |
| `README.md` | this |

## What you need

- E60 M5, MSS65 with V452, gearbox coded SMG
- RSA bypass
- E90 M3 DCT on stock software (DKG_90), no TCU flash, no TCU coding
- Longitudinal acceleration sensor wired to the DME the same way as the manual car ([wiring](#acceleration-sensor-wiring))
- [MSS6x Flasher](https://nam3forum.com/forums/forum/e9x-2008-2013/e9x-coding-tuning/11026-mss6x-flasher-now-released) by Terraphantm
- TunerPro, INPA/Tool32, NCS Expert

## Acceleration sensor wiring

On the SMG car the longitudinal acceleration comes from the SMG unit over CAN. With the SMG gone, nothing sends it, so DSC loses hill assist and shows the brake warning, and the DCT doesn't get the acceleration it needs. The manual M5 has the sensor wired straight to the DME, and the DME pins and the input are already there in V452. Wire it like the manual car and the patch does the rest (reads it on IGN, sends `0x1A3`, feeds `0x323`).

Sensor: **B49 Längsbeschleunigungssensor**, connector **X6249**. DME: **A6000**, connector **X60001**.

| B49 pin (X6249) | Signal | Wire | DME pin (X60001) | Notes |
|---|---|---|---|---|
| 3 | 5 V supply | 0.35 GE/GN (yellow/green) | 11 | 5 V sensor supply from the DME. Splice X14117 shares it with other sensors |
| 2 | GND | 0.35 BR/GN (brown/green) | 10 | Sensor ground. Splice X14048 shares it with other sensors |
| 1 | SENS_LBS (signal) | 0.35 GE/RT (yellow/red) | 9 | Analog acceleration signal into the DME |

```
 DME A6000 X60001                         B49 X6249
   pin 11  5V   ──── 0.35 GE/GN ─────────  pin 3
   pin 10  GND  ──── 0.35 BR/GN ─────────  pin 2
   pin 9   SENS ──── 0.35 GE/RT ─────────  pin 1
```

- Mount the sensor where the manual car has it, the right way round (it's directional).
- Check with Tool32/INPA after wiring: the acceleration should read about 0 standing still on flat ground and move when you brake or accelerate.
- The patch takes the value from this sensor path (IGN `0x29764`), so without the sensor the DSC and DCT get a constant value.

---

## How it works

The MSS65 has two MPC563s:

- **INJ** handles the PT-CAN messages used here, the DSC torque request decoder and the CAN send scheduler.
- **IGN** handles the torque structure, ASC/MSR intervention and the SMG module (clutch state, SMG torque logic).

They talk through a dual-port RAM (DPR).

```
               PT-CAN
  DCT ──0xB8 torque request──────┐
      ──0xBA gear / launch status─┤        INJ                         IGN
      ──0x1A2 input/output speed─┼──► receive slots ─► WRAP ─► can_dsc_eingr ──► ASC/MSR torque path
      ──0x3B1────────────────────┘        │        (caps/floors/                 (FUN_0004EEB8)
                                          │         blips/launch)
                                          └─► DPR export (seq + xor) ──► SMG status getter hooks
                                                                           0x321/0x322/0x323 rebuilt:
                                                                           clutch state, gear,
                                                                           m_kupp_max, acceleration
  DME ──0x1A3 acceleration (E9x format) ──► DSC (hill assist) + DCT
      ──0x34F handbrake released ─────────► DCT
      ──0x399 M symbol ───────────────────► cluster
```

A few rules I kept the whole way:

1. Program flash only. Calibration stays free so you can tune it like any V452.
2. Use what the DME already has. DCT torque requests go through the DSC intervention path, and the clutch state goes in through the SMG status getters.
3. If something goes wrong, fall back. Old DCT data gets dropped, a real DSC request always wins, and every function can be switched off on its own.

---

## CAN messages

### From the DCT

| ID | What's used | Notes |
|---|---|---|
| `0xB8` | B1hi+B2 = `md_soll` (12 bit, 0.5 Nm, `0x800` = none), ignition path. B3+B4lo = `md_soll_pos`, charge path. B4hi+B5 = clutch torque. B6 low = mode. B7<<4 \| B6>>4 = target speed × 2.5 rpm | Same layout as DSC `0xB6`. Field order checked against the MSS60 V762 decoder. |
| `0xBA` | B0 low = gear (1 N, 2 R, 3 P, 5..11 = 1st..7th). B3–B4 = output speed × 8. B7 bits 6–7 = `ST_RSTA` (launch: 10 prep, 01 active) | Launch bits found in the logs, match the PSB. |
| `0x1A2` | B0–B1 / 8 = input speed. B4–B5 − 2000 = output rpm | For clutch slip. |
| `0x3B1` | only logged | |

`0xB8` modes: 3 cap, 4 floor, 6 speed control (target rpm), 0xC clutch cap, 0 nothing.

### Sent by the DME (new or changed)

| ID | Content | Why |
|---|---|---|
| `0x1A3` | B0–B1lo = acceleration (12 bit, same as E60). B1hi+B2 = `0xFFF`. B3 = alive<<4. B4 = checksum. DLC 5 | E60 DSC reads B0–B1, the DCT wants alive and checksum (fixes CF21). |
| `0x34F` | B0 = `0xFD` (handbrake released), DLC 1, every 200 ms | DCT handbrake input (fixes CF25, needed for launch). |
| `0x399` | B4 bits 4–5 = M symbol (0 off, 1 on, 2 flashing) | Shows P in the cluster. |

### Rebuilt inside the DME (SMG status, IGN side)

| ID | Content |
|---|---|
| `0x321` | alive, input speed |
| `0x322` | alive, `kupp_st` (1 closed, 2/3 slipping), gear (1..7, 8 = R, 0 = N/P) |
| `0x323` | alive, acceleration (`smg_ax_mess`), `m_kupp_max` |

---

## INJ side

### Receiving the DCT

The V452 CAN driver sends incoming frames from its range mailboxes (controllers 0/1/2, objects 14/15) through a table at **INJ `0x2B224`**. Each entry is 20 bytes: message number, ID range, controller, object, callback. Some entries are unused (`0xFF` ranges), and I took four of them:

| Entry | ID | Controller / object | Callback |
|---|---|---|---|
| `0xCF` | `0x0B8` | 0 / 14 | `cb0` |
| `0xE5` | `0x0BA` | 2 / 15 | `cb1` |
| `0xE0` | `0x1A2` | 2 / 15 | `cb2` |
| `0xD8` | `0x3B1` | 1 / 15 | `cb3` |

All four go to one routine (`common`, INJ `0x4A5590`). It:

1. reads the frame with the driver's own `FUN_00017FE8(msg, buf)`
2. checks checksum and alive counter (seed = CAN ID)
3. stores the data and counters in the LOG block (`0x3FB930`, 128 bytes)
4. copies the frames over to IGN through the DPR at `0x1600720`, with a sequence number and an XOR check word

### Torque requests (WRAP)

The stock DSC decoder `FUN_0002D0FC` runs every 5 ms. It writes the DSC request into `can_dsc_eingr` (`0x3F88EB`, `+1` = `md_soll_pos`, `+3` = `md_soll`), and IGN turns that into ASC (cap) or MSR (floor).

The call at **INJ `0x2C25C`** now goes to WRAP (`0x4A5900`). WRAP:

1. puts back the last real DSC values, runs the stock decoder, saves the result
2. if DSC wants something, DSC wins and goes through unchanged
3. if not, takes the last valid `0xB8` (if it's not older than *request max age*)
4. turns the DCT mode into a DSC request:

| DCT mode | What happens | `can_dsc_eingr` |
|---|---|---|
| 3 / 0xC | `md_soll` and `md_soll_pos` as cap (0xC uses the lower one for both) | 2 = ASC |
| 4 | clamp to *floor max*, send as floor | 1 = MSR |
| 6 | floor = base + gain × (target − n), clamped | 1 = MSR |
| 0 / old | ramp out the last cap, then release | 2 → 0 |

Mode 6:

```asm
m6:  lbz   r9, 0x27(r31)        ; B7
     slwi  r9, r9, 4
     lbz   r10, 0x26(r31)       ; B6
     srwi  r10, r10, 4
     or    r9, r9, r10          ; target, 2.5 rpm/bit
     cmpwi r9, 0xFFF
     beq   nrm                  ; invalid
     mulli r9, r9, 5
     srawi r9, r9, 1            ; -> rpm
     lis   r10, hi(NINJ)
     lhz   r10, lo(NINJ)(r10)   ; engine speed
     subf  r9, r10, r9          ; error = target - n
     lha   r0, 0xE(r29)         ; gain (x/256 per 0.5 Nm)
     mullw r9, r9, r0
     srawi r9, r9, 8
     lha   r0, 0x10(r29)        ; + base
     add   r9, r9, r0           ; then clamp to floor min/max
```

Ramp out: after a cap ends, the cap goes up by *cap ramp-out rate* every call. Once both values are past *cap release level*, it lets go. The stock ASC exit ramp was way too slow (around 0.8 Nm per call), so I do it myself.

All the state is in the S3 block (`0x3FB8C0`, 64 bytes): saved DSC values, ramp state, age, counters, raw `0xB8`, mode, target, error, mode 6 floor and the launch counter.

### Launch limiter

At the end of WRAP. It runs when enable bit 64 is set, DSC isn't intervening, `0xB8` is fresh and `0xBA` B7 bit 7 (launch prep) is set:

```asm
     lhz   r9, 0x16(r29)        ; launch rpm (0 = DCT mode 6 target)
     ...
lc1: lis   r10, hi(NINJ)
     lhz   r10, lo(NINJ)(r10)
     subf  r9, r10, r9          ; launch rpm - n
     lha   r0, 0x18(r29)        ; gain
     mullw r9, r9, r0
     srawi r9, r9, 8
     lha   r0, 0x1A(r29)        ; + base, clamp to cap max / cap min
     ...
lc3: cmpwi r5, 2                ; DCT already capping? keep the lower value
     ...
lc6: li    r5, 2                ; ASC cap
```

Full throttle, the cap holds the engine at launch rpm. Let go of the lever, the flag drops and the normal ramp-out gives the torque back to the DCT.

### `0x1A3` in E9x format

- DLC at **INJ `0x2B72C`** changed from 2 to 5.
- Coding check at **INJ `0x2C044`** (only sends with manual gearbox) is `nop`, so it sends with SMG coding too.
- Send call at **INJ `0x2CFDC`** goes through `x1a3`, which adds offset, alive and checksum and then jumps to the stock send `FUN_00023BF0`:

```asm
x1a3: li    r0, 0xFF            ; B2..B4 default
      ...
      lbz   r11, 0x18(r12)      ; alive 0..14
      addi  r11, r11, 1
      cmplwi r11, 15
      blt   xa
      li    r11, 0
xa:   stb   r11, 0x18(r12)
      slwi  r0, r11, 4
      stb   r0, 3(r4)           ; B3 = alive << 4
      li    r10, 0x1A3          ; checksum = fold(ID + B0..B3)
      ...                       ; add B0..B3
      srwi  r0, r10, 8
      add   r10, r10, r0
      stb   r10, 4(r4)          ; B4
      b     FUN_00023BF0
```

Checksum is the same one the DCT checks with: `sum = ID + all bytes − checksum byte`, `chk = low + high`.

### `0x34F` handbrake

Sent from WRAP every 40 calls (200 ms) while the DCT is alive. It uses CAN object 12 on controller 2, which the DME also uses for `0x38E`/`0x3EF`. The patch borrows the object for one frame and sets the driver's "current message" byte (`0x3F8D52`) to something else, so the stock senders load their own ID again afterwards. Order: critical section, busy check `FUN_00018690`, set ID/DLC `FUN_00018444`, write and send `FUN_00023E58`.

### M symbol in P

In the `0x399` builder (`FUN_0045425C`), the instruction at **INJ `0x454380`** is now a call to `pblk`. When the DCT says P, the M symbol value (normally `mdrv_st` / `mdrv_tblink`) is swapped for the *M symbol mode* setting. The `0x399` checksum gets calculated after that, so it's still right.

---

## IGN side

The SMG module reads its status messages with a getter (`FUN_00024CF0`), called at **IGN `0x5CA78`, `0x5CEAC`, `0x5D498`** for `0x321`, `0x322`, `0x323`. Those calls go to the patch (`0x481AE0`). It:

1. copies the DPR block, checks sequence and XOR, counts good/bad/old
2. if there's no fresh DCT data within the *stale limit*, uses the stock getter (no SMG on the bus, so normal SMG timeout behaviour)
3. otherwise builds the SMG frame itself:

**Clutch state**, with hysteresis on slip = |n − input speed|:

```
closed  when slip < 60 rpm and input speed > 200 rpm
open    when slip > 150 rpm
kupp_st = 1 closed, else 2/3 from the clutch torque trend
```

**Gear:** `0xBA` 5..11 → 1..7, R → 8, N/P → 0.

**Acceleration:** `afz` (IGN `0x3F832A`, from the sensor) × 5/2 into `0x323` B1hi+B2 (12 bit, clamped, never `0xFFF`). `m_kupp_max` = setting × 5 Nm (600 Nm default).

Also on IGN:
- **IGN `0x29764`** `bne` → `nop`: acceleration comes from the analog sensor (`FUN_00029834`, `afz_ad`, offset learning `afz_a0_nv`) and not from SMG CAN.
- Program end pointer **IGN `0x1001C`** and CRC constants **IGN `0x30660`/`0x30668`** moved so the checksum covers the patch.

---

## DSC and DCT faults

| Problem | Cause | Fix |
|---|---|---|
| DSC hill assist off, brake warning light | E60 DSC gets `0x1A3` (acceleration) from the SMG | sensor wired to the DME, IGN sensor path on, `0x1A3` sent with any coding |
| DCT CF21 "Rohdaten Längsbeschleunigung, fehlerhafte Checksumme" | DCT wants the E9x 5-byte `0x1A3` with alive and checksum | send `0x1A3` in E9x format, E60 bytes stay the same |
| DCT CF25 handbrake timeout | E60 has no `0x34F` on PT-CAN | DME sends "released" (same value the DCT uses as substitute) |

---

## Launch control

I found the arming logic in the DCT firmware and checked it on the car. All of these have to be true:

| Condition | From |
|---|---|
| DSC fully off (mode 1 or 3) | `0x19E` |
| Manual, Drivelogic 6 | DCT |
| Lever pushed forward and held > 0.3 s | DCT |
| Engine warm flag | DME `0x1D0` (cat heating done) |
| Odometer ≥ 2000 km | cluster `0x330` |
| Handbrake signal valid and released | `0x34F` |
| Gearbox oil about 60–100 °C | DCT |
| Foot off the brake (brake cancels it) | DCT |

How: DSC off, manual, Drivelogic 6, push the lever forward and hold until the flag shows, full throttle, let go of the lever.

---

## Cluster

The E60 KOMB60 decodes the DCT's `0x1D2` (gear display). From its firmware (F2MC-16LX, decoder at `0xFC0000`):

- `MOTORSPORT` = wert_01 (stock M5 SMG) takes R, N, D + gear with mode 8 ("D1".."D7"), mode 2 (only the number, manual) and blank. **P is invalid.** The display goes blank and it stores **A3C6 / A569** ("Konsistenzfehler Getriebeposition"), which brings up **CC-ID 179**, the yellow cog.
- `MOTORSPORT` = wert_02 takes P, but then the DCT's D (mode 8) is invalid. Worse.
- The SMG never sent P at all. Checked in the SMG MK3 firmware: N `B4`, R `D2`, D `78`, blank `F0`.

Coding I used (block `0x3001` = `CC_ID_AKTIV`, one bit per CC-ID, checksum = XOR of the data bytes):

```
B 00300100,0010,4B,FF,FF,FB,FF,FF,FF,FF,F7,FF,FF,FF,FF,FF,FF,FF   (CSUM 43 -> 4B)
B 00300110,0010,FF,FF,FF,FF,FF,FF,FF,F7,FF,FF,FF,FF,FF,FF,FF,FF   (0x300117 FF -> F7 = CC 179 off)
```

Everything else stock (`0x300301` = `80`: GETRIEBE_ART ssg, MOTORSPORT wert_01; `0x300405` = `07`).

---

## XDF parameters

`V452_DCT_params.xdf`, open it on the full bin. Every change here needs a full flash. One call = 5 ms.

### General

| Parameter | Offset | Default | |
|---|---|---|---|
| Enable bits | `0x126004` | 127 | Add up: 1 caps, 2 floors, 4 blips (mode 6), 8 `0x34F`, 16 `0x1A3` E9x, 32 M symbol in P, 64 launch limiter, 128 N in P (keep off) |
| Request max age | `0x126006` | 5 calls | `0xB8` request dropped if no new frame comes in this time |

### Caps (upshifts, stall protection)

| Parameter | Offset | Default | |
|---|---|---|---|
| Cap ramp-out rate | `0x126008` | 20 Nm/call | How fast torque comes back after a cut. Higher = harder shift |
| Cap release level | `0x12600A` | 600 Nm | Lets go once the cap is above this |

### Floors and blips (downshifts)

| Parameter | Offset | Default | |
|---|---|---|---|
| Floor max (mode 4) | `0x12600C` | 300 Nm | Limit for fixed floors. `K_MD_MSR_MAX` still applies |
| Mode 6 gain | `0x12600E` | 0.15 Nm/rpm | floor = base + gain × (target − n). Higher = sharper blip |
| Mode 6 base | `0x126010` | 20 Nm | Floor when rpm is on target (friction) |
| Mode 6 floor max | `0x126012` | 200 Nm | Max blip torque |
| Mode 6 floor min | `0x126014` | −50 Nm | Min (engine above target) |

### Clutch state (IGN)

| Parameter | Offset | Default | |
|---|---|---|---|
| Slip to close | `0x382004` | 60 rpm | Below this the clutch counts as closed |
| Slip to open | `0x382006` | 150 rpm | Above this it's open again |
| Min input speed | `0x382008` | 200 rpm | Needed before it can be closed |
| m_kupp_max in `0x323` | `0x38200A` | 600 Nm | Clutch torque the SMG logic sees. Keep it above engine torque |
| Stale limit | `0x38200C` | 30 calls | After this it goes back to the stock getter |

### Launch

| Parameter | Offset | Default | |
|---|---|---|---|
| Launch rpm | `0x126016` | 4000 rpm | Held while the flag is on. 0 = use the DCT's mode 6 target |
| Launch gain | `0x126018` | 0.15 Nm/rpm | Lower it if rpm hunts |
| Launch base | `0x12601A` | 40 Nm | Cap at launch rpm. Raise it if rpm drops |
| Launch cap max | `0x12601C` | 500 Nm | Limit below launch rpm |
| Launch cap min | `0x12601E` | −50 Nm | Limit above launch rpm |

### Display

| Parameter | Offset | Default | |
|---|---|---|---|
| M symbol mode in P | `0x126005` | 2 | 0 stock, 1 on, 2 flashing. Needs enable bit 32 |
| N in P send offset | `0x12603C` | 20 | Only for N in P, which is off |

---

## Stock V452 parameters with the DCT

| Group | Examples | What happens |
|---|---|---|
| Engine calibration | fuel, ignition, knock, VANOS, lambda, torque model, pedal, idle, limiters | Same as before. Calibration isn't touched. Where there are SMG/manual versions, SMG is used |
| SMG shift / clutch control | `K_SMG_N_ZIEL_*`, `K_SMG_KUPP_*`, drive-off, `K_SMG_TESTMODE` | Not active. These react to SMG requests in `0x301`–`0x307` and the DCT doesn't send them |
| SMG status users | Kraftschluss, clutch max torque, acceleration | Active, fed with DCT data. Tune with the patch XDF, not `K_SMG_KSCHLUPF_*` |
| Gear-based maps | `KL_MD_KUPP_MAX_GANG`, `KL_MD_NBEGR_GANG`, overrun cut-off / anti-jerk per gear, `K_J_FZG_SMG_1..7` | Active with the real DCT gear |
| Gear detection | `KL_GANG_I_SMG`, `K_GANG_I_HA_SMG` (3.62), `K_GANG_I_RW_SMG` | Leave stock. DCT ratios with the 3.15 diff land in the same 1st–7th slots |
| ASC/MSR | `K_MD_ASC_ENABLE`, `K_MD_MSR_ENABLE` (keep at 1), `K_MD_MSR_MAX` (300 Nm), ramps, wait times | Shared by DSC and DCT. Change them and DSC traction control changes too |
| DKG | `K_J_FZG_DKG_*` | Not used (coding is SMG) |

---

## Memory map

Full bin, 5 MB:

| File range | |
|---|---|
| `0x000000`–`0x06FFFF` | INJ program, internal flash |
| **`0x070000`–`0x07FFFF`** | **INJ calibration, untouched** |
| `0x080000`–`0x27FFFF` | INJ program, external flash (CPU `0x400000`–`0x5FFFFF`, file = CPU − `0x380000`) |
| `0x280000`–`0x2EFFFF` | IGN program, internal flash |
| **`0x2F0000`–`0x2FFFFF`** | **IGN calibration, untouched** |
| `0x300000`–`0x4FFFFF` | IGN program, external flash |

Everything that's different from stock:

| CPU | File offset | What |
|---|---|---|
| INJ | `0x01001C` | program end pointer |
| INJ | `0x02B2A0`–`0x02B467` | receive table entries |
| INJ | `0x02B72C` | `0x1A3` DLC 5 |
| INJ | `0x02C044` | `0x1A3` with any coding |
| INJ | `0x02C25C` | hook to WRAP |
| INJ | `0x02CFDC` | hook to `x1a3` |
| INJ | `0x0309CA` | CRC constant |
| INJ | `0x0D4380` | hook in `0x399` builder |
| INJ | `0x125590`–`0x1258C7` | receive callbacks, DPR export |
| INJ | `0x125900`–`0x125EE3` | WRAP |
| INJ | `0x126000`–`0x12603F` | parameters, CAN descriptors |
| INJ | `0x126040`–`0x1262F3` | `x1a3`, `pblk` |
| INJ | `0x126400` | info text (encoded) |
| IGN | `0x29001C` | program end pointer |
| IGN | `0x2A9764` | sensor path |
| IGN | `0x2B0668` | CRC constant |
| IGN | `0x2DCA78`, `0x2DCEAC`, `0x2DD498` | SMG getter hooks |
| IGN | `0x381AE0`–`0x381EEF` | SMG status rebuild |
| IGN | `0x382000`–`0x382013` | parameters |
| IGN | `0x382400` | info text (encoded) |

RAM (CCP DAQ list 2 area, not used in the car):

| Address | CPU | |
|---|---|---|
| `0x3FB8A0` | INJ | send state (`0x34F`, `0x1A3` alive, M symbol) |
| `0x3FB8C0` | INJ | S3, torque request state |
| `0x3FB930` | INJ | LOG, received DCT frames and counters |
| `0x3FADB8` | IGN | DKS2, DPR copy, clutch state, rebuilt SMG frames |

---

## Flashing

| Change | Flash |
|---|---|
| New patch version or anything in the patch XDF | full flash (MSS6x Flasher does the program checksum) |
| Maps in the normal V452 XDF | calibration flash, the patch stays |
| Calibration from another bin | copy only the calibration areas into the patched full bin |

Don't full flash a stock bin over it, that takes the patch out.

---

## Testing

Before anything went in the car:

- The patch is assembled from PowerPC source and written into the stock V452 bin. Every address gets checked against the stock bytes first. If one doesn't match, the build stops, so a wrong bin or a wrong address can't end up in a flashable file.
- I ran the real DME code from both processors in an emulator, starting from RAM read out of the car, and fed it DCT frames (`0xB8`, `0xBA`, `0x1A2`). It checked the torque limits going into ASC/MSR, what gets sent and when, and the `0x1A3` checksum against the DCT's algorithm. A new stage only went out if all the older results still came out the same.
- Then on the car with logs and fault memory reads.

---

## Logging

No CCP, no CAN interface. Everything went over the normal diag cable with EDIABAS (`api32.dll`), the same jobs Tool32 uses, in a loop at about 10 Hz.

What I wanted: see what the DCT asks for, what the DME does with it and what the other modules think, all at the same time. Then change one thing and see if it fixed it.

How: the patch keeps its state in free RAM (the CCP DAQ area). That's the last raw frame of each DCT message, the counters, the decoded requests, the state machine and the rebuilt SMG frames. Reading that with `ram_lesen` on the DME (IGN at address + `0x800000`, 100 bytes max per read) gives you a CAN trace and the internal state in one log. No extra hardware needed.

| Logged | From | Answered |
|---|---|---|
| Raw DCT frames `0xB8`, `0xBA`, `0x1A2`, `0x3B1` | DME RAM (LOG / DKS2) | Layout, request modes, gear coding, launch bits |
| Engine speed, input speed, slip | DME RAM | Clutch close/open thresholds |
| `md_kupp_ist`, `md_kupp_ist_ve` | IGN RAM | Does the SMG logic take the rebuilt clutch state |
| State (DSC / cap / floor / ramp-out / speed / launch), cap and floor, mode 6 target and error | S3 block | Shift cuts, blips, ramp-out, DSC priority |
| `can_dsc_st`, engine state, cat heating, oil temp | DME RAM | Launch conditions on the DME side |
| All DCT actual values (`STATUS_ISTWERTE_LESEN`) every 2 s | DCT | Launch conditions, lever, handbrake, temperatures |
| DCT `FS_LESEN` / `IS_LESEN` before and after each test | DCT | Which messages the DCT doesn't like (CF21, CF25) |
| Received `0x1D2`, gear display state, coding byte | Cluster RAM (`speicher_lesen`) | Why P is blank, where A3C6/A569 come from |
| `STATUS_CHECKCONTROL_LESEN`, cluster faults | Cluster | Which CC-ID turns on the yellow cog |

Every test was the same routine with key markers in the log (DSC off, mode set, brake, full throttle, release), so I could line up the log with what was happening in the car.

---

## How it went

**Stage 1, getting the DCT in.** V452 knows nothing about the DCT, but the receive table had free slots. The DCT frames go in there, get checked and logged, and get passed to IGN through the DPR.

**Stage 2, clutch state.** The SMG logic needs Kraftschluss or idle and anti-jerk act up. Logs of pull-aways and holding 1st showed the slip between engine and DCT input speed. A hysteresis on that slip, written into rebuilt `0x321`–`0x323`, gave a clean closed/open (IGN rev C).

**Stage 3a–3b, torque requests.** Logs showed caps (mode 3) on upshifts and floors on downshifts. E9x `0xB8` is the same layout as the E60 DSC `0xB6`, so it goes straight into the DSC intervention path, and shift cuts worked. I had the field order wrong at first from the docs (`md_soll` is ignition, `md_soll_pos` is charge) and fixed it by checking the MSS60 V762 decoder. The stock ASC exit ramp was far too slow, so the patch does its own ramp-out.

**Stage 3b, blips.** Downshifts use mode 6 with a target rpm. A simple P controller (gain + base) turns the rpm error into a floor.

**Stage 3c–3e, DSC and acceleration.** Hill assist and the brake light didn't work because the E60 DSC wants `0x1A3` from the SMG. I wired the sensor to the DME, turned on the IGN analog path (with SMG coding it was stuck at the 389 default) and made `0x1A3` send with any coding. Brake light gone. The same acceleration goes into `0x323` for the oil level measurement.

**Cluster.** P didn't show. I went through the KOMB60 firmware (F2MC-16LX in Ghidra) and the SMG MK3 firmware (C166 in Ghidra). The SMG never sent P, and the wert_01 decoder treats the DCT's P as invalid. That's what sets A3C6/A569 and the yellow cog (CC-ID 179). I first read the fault table wrong and thought it was a port input check, then decoded the records properly. Fix: CC-ID 179 off in `CC_ID_AKTIV`, and the checksum turned out to be XOR of the block.

**Stage 3g, DCT faults.** Used the DCT fault memory and the DCT firmware. CF21 is the `0x1A3` checksum: the DCT wants the E9x 5-byte frame, and I took the layout, alive and checksum from its receive code. CF25 is missing `0x34F`, and its decoding also came from the DCT (1 released, 2 applied, 3 invalid). Both fixed, and the DCT fault memory is clean.

**Launch.** Brake plus throttle only went to about 1400 rpm. That's the DCT's stall protection (mode 3, about 145 Nm), because launch wasn't armed. I found the arming function in the DCT firmware, then logged the DME and all DCT values together. That showed what was missing: the lever has to be pushed forward and held, and the brake actually cancels it. The launch flag (`0xBA` B7 bit 7) came out of those logs.

**Stage 3h:** launch limiter and the M symbol mode setting.

**Stage 3i:** N in P (didn't work, see below) and the info text in both processors.

Tools: Ghidra (PowerPC for DME and DCT, [C166 module](https://github.com/esaulenka/Ghidra_C166) for the SMG, [F2MC-16LX module](https://github.com/mehmooda/F2MC-16LX) for the cluster), PowerPC assembler/disassembler, a CPU emulator, EDIABAS, the MSS60 PSB and V762 bins as reference, NCS SWT files for the cluster coding.

---

## What didn't work

**N in P.** The DME picked up the DCT's `0x1D2` and sent its own N frame in between the DCT's frames (they come every 200 ms). For the cluster to accept two senders I had to code its EGS checksum/alive checks off (`0x300405` = `01`). With that, the cluster showed no gear at all and the DSC light came on. The code is still there but switched off (enable bit 128). To really fix it you'd need a CAN gateway on the DCT lines or a cluster patch.

---

## Still to do

- Road test and tune the launch limiter (rpm, gain, base).
- Check the oil level measurement now that it gets acceleration.
- Read RAM to see whether the DME takes the gear from the rebuilt `0x322` or works it out from the ratio. Either way it ends up with the same gear.
- Maybe a CAN gateway for N/P on the display.

---

## Shout-out

Big thanks to **Terraphantm** for the [MSS6x Flasher](https://nam3forum.com/forums/forum/e9x-2008-2013/e9x-coding-tuning/11026-mss6x-flasher-now-released). Every full read, full flash and calibration flash in this project was done with his tool. Without it, none of this would have made it onto the car. Go thank him on Nam3Forum.

---

## Licence

Created by **ZiMMie** for the M5Board and Nam3Forum communities.

This project is intended to make this information freely accessible to everyone, promoting knowledge sharing, collaboration, and learning. Everyone is welcome to use, build upon, and contribute their own knowledge to the project.

This project is provided for **non-commercial use only**. It may not be sold, used for commercial purposes, or redistributed for profit.

The bin is for V452 only. You can copy your own calibration into the calibration areas (see [Flashing](#flashing)).

---

## Support

Value for Value. If this helped you out, give back what you think it was worth. A coffee, a log, a bug report, whatever.

[![Buy Me a Coffee](https://img.shields.io/badge/Buy%20me%20a%20coffee-Value%20for%20Value-FFDD00?style=for-the-badge&logo=buymeacoffee&logoColor=black)](https://buymeacoffee.com/e6xdonate)
