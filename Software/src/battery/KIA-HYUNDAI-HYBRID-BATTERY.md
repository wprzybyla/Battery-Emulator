# Kia / Hyundai Hybrid (HEV / PHEV) battery

Driver for the Mobis traction battery used in the **Kia Ceed PHEV** (96S, 300–404 V, 8.9 kWh)
and shared, with the same BMU protocol, by the **Hyundai Santa Fe PHEV** and the
**Hyundai Ioniq / Kia Niro HEV/PHEV** (56S). The pack is auto-detected as 96S PHEV or 56S HEV
from the measured pack voltage.

The CAN reverse-engineering this driver is based on was verified on a real Kia Ceed PHEV 96S
(capture `logceed5`): https://github.com/wprzybyla/Hyundai-Kia-EV-HEV-PHEV

## Why a state machine is needed

The BMU **will not close its contactors on its own**. In the car it waits for a sequence of
frames from the HCU/HPCU; without them it either stays asleep or shuts back down after ~1 minute.
The driver emulates that vehicle side with a small state machine:

```
IDLE ──▶ KL15 ──(~300 ms)──▶ PRECHARGE ──(~1.5 s)──▶ ACTIVE
```

| State | What it sends | What the BMU does |
|-------|---------------|-------------------|
| `IDLE` | (transient, auto-starts into KL15) | — |
| `KL15` | `0x523` ignition/HCU ready | Wakes, listens |
| `PRECHARGE` | `0x200` D4 precharge bit + `0x2A1` voltage ramp | Charges the DC-link capacitors |
| `ACTIVE` | `0x2F0` contactor bits + steady `0x200`/`0x2A1`/`0x523` | Contactors closed, HV on the output |

If the CAN stream stops (e.g. the emulator pauses on a fault), the BMU times out, opens the
contactors and returns to a fault state; a KL15 OFF→ON restart is required to recover.

All transmitted frames are 8 bytes on a single 500 kbps 11-bit bus.

## Transmitted frames (emulator → BMU)

### `0x523` — Ignition / HCU ready (100 ms)

| Byte | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|------|---|---|---|---|---|---|---|---|
| | `0x60` IGN on | – | `0x60` HCU ready | – | – | – | – | alive (2 bit) |

No CRC. **Without this frame the BMU ignores everything else.**

### `0x200` — HV request (10 ms)

| Field | Byte | Bits | Value |
|-------|------|------|-------|
| Precharge | 4 | b7 | `0x80` during PRECHARGE, `0x00` otherwise |
| Constant | 5 | — | always `0x30` |
| Alive | 6 | b1–b4 | 0–15, +1 per frame |
| CRC8 | 7 | all | polynomial `0x01` |

Contactors are **not** controlled here — byte 5 stays `0x30` in every state.

### `0x2A1` — DC-link voltage (10 ms)

| Byte | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|------|---|---|---|---|---|---|---|---|
| | `0x00` | – | – | – | – | – | voltage | `0x02` |

Byte 6 is a **logical** value for the BMU state machine, **not a real measurement**:
* `PRECHARGE`: monotonic ramp `0x30 → 0x45` (+1 every 10 ms across the ~1.5 s window)
* `ACTIVE`: `0xFF`
* otherwise: `0x00`

No CRC.

### `0x2F0` — HV enable / contactors (10 ms)

| Field | Byte | Bits | Value |
|-------|------|------|-------|
| Contactor 1 | 0 | b0 | `0x01` (ACTIVE only) |
| Contactor 2 | 6 | b6 | `0x40` (ACTIVE only) |
| Alive | 6 | b0–b1 | 0–3, +1 per frame |
| CRC8 | 7 | all | polynomial `0x01` |

### CRC8

Used only on `0x200` and `0x2F0`. Polynomial `0x01`, byte 7 zeroed before calculation, computed
over all 8 bytes:

```
crc = 0; data[7] = 0
for i in 0..7:
  crc ^= data[i]
  for j in 0..7:
    crc = (crc & 0x80) ? ((crc << 1) ^ 0x01) : (crc << 1)
    crc &= 0xFF
data[7] = crc
```

## Received frames (BMU → emulator)

* **`0x5AE`** — keep-alive + interlock. `data[1] & 0x02` non-zero ⇒ HV interlock loop open
  (fault); zero ⇒ OK. Also refreshes `CAN_battery_still_alive`.
* **`0x7EC`** — multi-frame UDS response to the `0x7E4` poll (`02 21 XX`, PID 1–5). Carries pack
  voltage/current, min/max/module temperatures, per-cell voltages (96 cells, ×20 mV) and SoC.
  The driver sends the ISO-TP flow-control ack (`30 00 …`) when it sees the first frame.

## Notes

* The pack has an internal precharge circuit (PRA); the emulator does not need to precharge the
  DC-link itself. The `0x2A1` ramp is what tells the BMU to run its own precharge.
* DTC reset is supported (service `0x14`, sent on `0x7E4` when the user requests it).
