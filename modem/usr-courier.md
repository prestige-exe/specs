# USR Courier AT Command Set

by U.S. Robotics, Inc. (Skokie, Illinois)
Documented in the "Courier High Speed Modems User's Manual" and the
"USR Courier Dual Standard Reference" published with each firmware
revision from 1986 onward. The Courier line began with the Courier
2400 in 1985, gained HST (High Speed Technology) at 9600 bps in 1986,
and culminated in the Courier V.Everything (V.34, then V.90, then
V.92) in the late 1990s. The Dual Standard variants supported both
HST and V.32bis (or V.34) modulations and could fall back between
them.

The Courier extended the Hayes AT command set with a large number of
proprietary commands prefixed by `&`, `\\`, `%`, and `*`. The HST
modulation, USR's asymmetric 9600/450 bps proprietary scheme, required
its own configuration commands. The S register space above S38 was
extended well past the Hayes-standard S0 through S12.

This document covers the commands unique to or significantly extended
by the Courier line. Base AT commands shared with the Hayes spec are
in `hayes-at.md`.

## Connection-state commands

### &A – ARQ result codes

| Setting | Meaning |
| --- | --- |
| `&A0` | ARQ result codes disabled. `CONNECT 14400` is reported regardless of error correction. |
| `&A1` | ARQ result codes enabled. The string `/ARQ` is appended on a reliable connect: `CONNECT 14400/ARQ`. |
| `&A2` | `&A1` plus modulation reporting: `CONNECT 14400/ARQ/V32/LAPM`. |
| `&A3` | `&A2` plus compression and protocol detail: `CONNECT 14400/ARQ/V32/LAPM/V42BIS`. |

`&A3` is the setting BBS sysops used so the modem reported exactly
what kind of link was running, which mattered when diagnosing why
ZModem transfers were running at half speed.

### &B – serial port speed handling

| Setting | Meaning |
| --- | --- |
| `&B0` | DTE rate floats with DCE rate. The COM port speed changes after CONNECT to match the line speed. |
| `&B1` | DTE rate is locked. The COM port stays at whatever rate it was opened at, regardless of DCE rate. Hardware flow control mandatory. |
| `&B2` | DTE rate is locked during ARQ connections, floats during non-ARQ. |

`&B1` was the canonical setting for a properly configured BBS modem.
With a locked DTE rate of 38400 or 57600 bps, the modem buffered data
and the FOSSIL driver delivered bytes as fast as the COM port could
take them, regardless of whether the modem-to-modem link was running
at 2400, 9600, or 14400 bps.

### &H – transmit data flow control

| Setting | Meaning |
| --- | --- |
| `&H0` | Flow control disabled. |
| `&H1` | Hardware (CTS) flow control. Modem drops CTS to stop the computer from sending. |
| `&H2` | Software (XON/XOFF) flow control. Modem sends 0x13 to stop, 0x11 to resume. |
| `&H3` | Both `&H1` and `&H2`. |

### &I – software flow control inbound

| Setting | Meaning |
| --- | --- |
| `&I0` | Disabled. |
| `&I1` | XON/XOFF on the data being sent from the computer; the modem acts on the codes and passes them to the remote. |
| `&I2` | XON/XOFF acted on locally; the codes are stripped and not sent to the remote. |

For file transfers (ZModem, YModem-G) `&I0` is required because the
binary data contains 0x11 and 0x13 bytes that would otherwise be
swallowed.

### &R – receive data flow control (CTS/RTS)

| Setting | Meaning |
| --- | --- |
| `&R0` | Modem ignores RTS. CTS tracks RTS in command mode. |
| `&R1` | Modem ignores RTS. CTS always on in data mode. |
| `&R2` | Modem uses RTS for hardware flow control of received data. The computer drops RTS to stop the modem from sending. |

For locked-rate operation with hardware flow control, `&H1 &R2` is
the pair.

### &K – data compression

| Setting | Meaning |
| --- | --- |
| `&K0` | Data compression disabled. |
| `&K1` | Auto: enable compression if the link is ARQ, disable otherwise. |
| `&K2` | Enable but only if also forced to ARQ. |
| `&K3` | Auto with selective disable: compression on for the link, but disabled at runtime if the modem detects already-compressed data. |

`&K3` was the Courier-specific equivalent of `%C1` plus the runtime
heuristic that disabled MNP 5 on incompressible streams. The
heuristic in `&K3` looked at the size of compressed output frames
relative to their inputs and disabled the compressor for a few
seconds when output grew larger than input.

### &M – ARQ mode and error correction

| Setting | Meaning |
| --- | --- |
| `&M0` | Normal mode (no error correction). |
| `&M1` | Reserved. |
| `&M2` | Reserved. |
| `&M3` | Reserved. |
| `&M4` | Normal/ARQ: try ARQ first, fall back to normal. |
| `&M5` | ARQ only: require ARQ, hang up if not available. |

ARQ in Courier terminology covers both LAPM (V.42) and MNP. The
preference order is V.42 first, then MNP. `&M4` is the Courier
equivalent of Hayes `\N3`.

### &N – fixed link speed (ceiling)

`&N0` is variable: the modem negotiates the highest mutually
supported rate. `&N1` through `&N16` lock the connect rate to a
specific value:

| Setting | Speed |
| --- | --- |
| `&N0` | Variable (auto-negotiate) |
| `&N1` | 300 bps |
| `&N2` | 1200 bps |
| `&N3` | 2400 bps |
| `&N4` | 4800 bps |
| `&N5` | 7200 bps |
| `&N6` | 9600 bps |
| `&N7` | 12000 bps |
| `&N8` | 14400 bps |
| `&N9` | 16800 bps |
| `&N10` | 19200 bps |
| `&N11` | 21600 bps |
| `&N12` | 24000 bps |
| `&N13` | 26400 bps |
| `&N14` | 28800 bps |
| `&N15` | 31200 bps |
| `&N16` | 33600 bps |

On V.Everything firmware, the table extends to 56000 and 64000 bps
for V.90 and the (never-deployed) 64k pulse-code modes.

### &U – minimum link speed (floor)

Same table as `&N` but specifies the lowest acceptable connect
rate. If the line will not support the floor, the modem hangs up
rather than connecting at a slower speed. `&U0` disables the floor.

A BBS configured for 9600 bps minimum would set `&U6 &N0` so calls
that would only train up to 2400 bps would be dropped rather than
tying up a port at low speed.

### &W – store configuration

`&W0` stores the active configuration as user profile 0. `&W1`
stores it as user profile 1. The Courier supports two stored
profiles plus the factory defaults (`&F0` through `&F3`).

`&F0` is the basic factory default. `&F1` is the "BBS template"
factory default with `&B1`, `&H1`, `&I0`, `&R2`, `&K3`, `&M4`, `X4`,
`&C1`, `&D2` and a 60-second carrier wait. `&F2` and `&F3` are
profiles tuned for different terminal-software conventions.

`AT&W` writes the current settings to NVRAM. The Y register (`Y0` /
`Y1`) selects which profile is loaded on power-on or on ATZ.

## S registers above S12

The Courier extends the S register space considerably. Selected
Courier-specific registers:

| Reg | Range | Default | Meaning |
| --- | --- | --- | --- |
| S13 | bitmask | 0 | Bit flags. Bit 0: reset on DTR drop. Bit 1: auto-answer in originate mode. Bit 2: hang up rather than disconnect on +++. Bit 3: enable status display lights. Bit 4: protected configuration. |
| S15 | bitmask | 0 | HST-specific tuning. Bit 0: disable HST. Bit 1: disable V.32. Bit 3: disable trellis encoding on V.32. Bit 4: disable MNP. Bit 7: disable LAPM. |
| S22 | 0..255 | 17 | XON character. Default 0x11. |
| S23 | 0..255 | 19 | XOFF character. Default 0x13. |
| S27 | bitmask | 0 | Connect-mode bits. Bit 0: enable ITU-T modes (V.32, V.34) when otherwise disabled by S15. Bit 2: disable V.32 auto-mode. Bit 3: disable 2100 Hz answer tone (use 2225 Hz Bell). Bit 4: enable V.21 (300 bps CCITT) instead of Bell 103. Bit 6: disable V.42 detect phase. |
| S32 | 0..255 | 17 | XON in flow control toward DCE. |
| S34 | 0..255 | 0 | Reserved. |
| S38 | 0..255 | 0 | Delay (seconds) before forced hangup on ATH. Allows pending data to drain. |

S15 and S27 are the registers that BBS sysops modified when diagnosing
HST/V.32 interoperability problems with other modems. Setting `S15=8`
to disable trellis encoding was a frequent workaround for noisy
inter-LATA lines.

## The canonical Courier init string

```
AT &F1 &B1 &H1 &I0 &R2 &K3 X4 S7=60
```

Read as:

- `&F1`: load the BBS template profile (locked DTE, hardware flow,
  ARQ auto, compression auto).
- `&B1`: lock the DTE rate. Redundant after `&F1` but explicit.
- `&H1`: hardware (CTS) flow control on outbound DTE data.
- `&I0`: no software flow control on inbound DTE data (so binary
  file transfers don't drop bytes).
- `&R2`: hardware (RTS) flow control on inbound from modem.
- `&K3`: compression auto with runtime disable.
- `X4`: full Hayes result codes (NO DIALTONE, BUSY, CONNECT speed).
- `S7=60`: wait 60 seconds for carrier after dial.

The string exists in this exact form (with minor variations) in
hundreds of BBS configuration files from 1991 through 1997. The
reason it was so universal is that USR's `&F1` defaults already
match what a BBS needed; the rest of the string just re-asserts the
defaults so a botched NVRAM does not produce a broken init.

For HST-specific Couriers the string commonly added `B0 S15=128`
(force HST, disable V.32) or its inverse for V.32-only operation
against non-HST callers.

## The HST modulation

HST is asymmetric: the higher-bandwidth direction runs at 9600,
14400, or 16800 bps depending on firmware; the back channel runs at
450 bps (300 baud trellis-encoded). HST is half-duplex on a single
carrier but the modem reverses the bandwidth assignment automatically
when the DTE has data to send the other way. This was tuned for BBS
traffic where the file-transfer direction is usually one-way and the
other direction only carries ACKs.

HST predates V.32 (and is incompatible with it) and was USR's main
selling point in the late 1980s. Dual Standard models added V.32 and
later V.32bis so they could talk to non-USR modems at high speed.

By the V.34 era, HST was a legacy compatibility feature; by V.Everything
it was effectively retired.

## NRAM and ATI

`ATI4` dumps the active configuration. `ATI5` dumps the contents of
NVRAM. `ATI6` dumps a link-quality and statistics report from the
most recent call: connect rate, retrains, error corrections, blocks
sent/received, compression ratio. `ATI7` and higher dump firmware
identification strings.

`ATY` (without a number) is the NVRAM-stored Y setting; `Y0` makes
profile 0 the power-on default, `Y1` makes profile 1 the default.

## References

- U.S. Robotics, "Courier HST and Courier HST Dual Standard High
  Speed Modems User's Manual", multiple revisions 1987 through 1995.
- U.S. Robotics, "Courier V.Everything Modem Reference Manual", 1996.
- U.S. Robotics, "Courier V.34 Reference", 1994.
- BBS Documentation Project archives of BBS-era USR init strings,
  including the `Modem.lst` file maintained by Robert Schenot and
  distributed via FidoNet.
