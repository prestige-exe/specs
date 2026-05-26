# Hayes AT Command Set

by Dale Heatherington / Hayes Microcomputer Products
Originally introduced with the Hayes Smartmodem 300 (1981) and the
SmartCom modem suite. Formalised into the "Standard AT Command Set"
across the industry by the mid-1980s and codified by ITU-T as
recommendation V.250 (1995). Earlier work appeared as ANSI/TIA/EIA-602
("Data Transmission Systems and Equipment – Serial Asynchronous
Automatic Dialling and Control").

The AT command set is the protocol every dial-up modem ever made
spoke. Every BBS terminal program issued AT commands to bring up the
modem, dial a number, manage the connection, and tear it down. The
"AT" stands for "ATtention" – it is the prefix that tells the modem
the following characters are commands, not data.

## States

A Hayes modem is in one of two states:

- **Command mode**: bytes from the computer are interpreted as
  commands. The modem responds with result codes.
- **Data mode** (also "on-line"): bytes from the computer are sent
  over the phone line; bytes from the phone line are delivered to
  the computer.

The modem starts in command mode after power-on. Issuing `ATD` (dial)
or `ATA` (answer) and getting a successful CONNECT moves it into data
mode. The escape sequence (`+++`, see below) moves it back.

## Command format

```
AT<command> [<command> ...] <CR>
```

- Every command line starts with "AT" (or "at").
- One line may contain multiple commands concatenated, no separator
  needed (though spaces are tolerated and stripped).
- The line ends with a CR (0x0D). LF is ignored.
- Commands are case-insensitive.
- Numeric parameters default to 0 if omitted (so `ATE` is `ATE0`).

The modem echoes the typed command back (if echo is on) and then
replies with a result code on its own line.

## Result codes

In numeric form (`ATV0`):

| Code | Meaning |
| --- | --- |
| 0 | OK |
| 1 | CONNECT |
| 2 | RING |
| 3 | NO CARRIER |
| 4 | ERROR |
| 5 | CONNECT 1200 |
| 6 | NO DIALTONE |
| 7 | BUSY |
| 8 | NO ANSWER |
| 9 | CONNECT 0600 |
| 10 | CONNECT 2400 |
| 11 | CONNECT 4800 |
| 12 | CONNECT 9600 |
| ... | (vendor-specific extensions, including all the V.32bis / V.34 codes) |

In verbose form (`ATV1`, the default), result codes are the equivalent
English strings: `OK`, `CONNECT`, `RING`, `NO CARRIER`, etc., each
preceded and followed by CR LF.

## Core commands

### Dialling and call control

| Command | Meaning |
| --- | --- |
| `ATD<number>` | Dial the number that follows; modifiers `T` (tone), `P` (pulse), `,` (pause), `W` (wait for dial tone), `@` (wait for silence), `!` (hook flash), `;` (return to command mode after dial) |
| `ATA` | Answer immediately |
| `ATH` or `ATH0` | Hang up |
| `ATH1` | Off-hook |
| `ATO` or `ATO0` | Return to data mode (after `+++`) |

### Basic configuration

| Command | Meaning |
| --- | --- |
| `ATZ` | Reset modem to power-on (or stored) profile |
| `AT&F` | Reset modem to factory defaults |
| `AT&W` | Save current configuration as power-on profile |
| `ATE0` / `ATE1` | Echo off / on |
| `ATQ0` / `ATQ1` | Quiet off (send result codes) / on (suppress them) |
| `ATV0` / `ATV1` | Numeric result codes / verbose result codes |
| `ATM0` / `ATM1` / `ATM2` / `ATM3` | Speaker off / on until carrier / always on / on during dialling |
| `ATL0`..`ATL3` | Speaker volume |
| `ATX0`..`ATX4` | Result-code level (0 = OK/CONNECT/RING/NO CARRIER/ERROR only; 4 = full set with NO DIALTONE, BUSY, CONNECT speed) |
| `ATS<r>=<n>` | Set S-register r to value n |
| `ATS<r>?` | Query S-register r |

### Inquiry

| Command | Meaning |
| --- | --- |
| `ATI` or `ATI0` | Product code |
| `ATI1` | ROM checksum |
| `ATI2` | ROM checksum test |
| `ATI3` | Firmware revision |
| `ATI4` | Settings dump |
| `AT&V` | View active and stored profiles |

## S-registers

Modem behaviour is parameterised by an array of "S-registers".
Standardised S-registers:

| Reg | Default | Meaning |
| --- | --- | --- |
| S0 | 0 | Rings before auto-answer (0 = disabled) |
| S1 | 0 | Ring counter (read-only) |
| S2 | 43 | Escape character (default '+') |
| S3 | 13 | CR character |
| S4 | 10 | LF character |
| S5 | 8 | BS character |
| S6 | 2 | Wait time (s) before dialling |
| S7 | 30 | Wait time (s) for carrier after dial |
| S8 | 2 | Comma pause duration (s) |
| S9 | 6 | Carrier-detect response time (1/10 s) |
| S10 | 14 | Carrier-loss disconnect delay (1/10 s) |
| S11 | 95 | Tone-dial duration (ms) |
| S12 | 50 | Escape guard time (1/50 s) |

Vendor S-registers (above S12) vary; consult the modem's manual.

## Escape sequence

To return to command mode while online without losing the connection:

```
   (silence for S12/50 seconds)
   +++
   (silence for S12/50 seconds)
```

The modem responds with `OK`. The escape character is S2 (default
`+`); the guard time is S12 (default 50/50 s = 1 second). The double
guard time is what prevents data containing literal `+++` from being
misinterpreted as an escape – legitimate data has no second of silence
around it.

Some BBSes and file-transfer protocols deliberately send `+++` in
data; the guard time is what keeps that safe.

## Extended commands

After AT command set basics, modem vendors added extensions:

- **`&` commands** – Hayes extended set: `&C1` (DCD follows carrier),
  `&D2` (DTR drop disconnects), `&K3` (RTS/CTS flow control), `&Q5`
  (auto-reliable / negotiate MNP4 fallback to V.42), etc.
- **`\` commands** – flow control and error correction selection:
  `\Q0`..`\Q3` (flow control), `\N0`..`\N5` (error correction level).
- **`%` commands** – data compression: `%C0` (none), `%C1` (MNP5),
  `%C2` (V.42bis), `%C3` (both).
- **`-` commands** – generally vendor-specific extensions, including
  the V.92 features.
- **`+` commands** – ITU-standardised extensions (`+MS=` for modulation
  selection, `+IPR=` for DTE rate, `+FCLASS=` for class).

## Typical BBS init string

```
ATZ
AT&F E1 V1 Q0 X4 S0=0 S7=60 S11=55 &C1 &D2 &K3 \N3 %C1
```

Translation:

- `ATZ` – reset to default profile.
- `AT&F` – then factory defaults again, in case the stored profile
  was odd.
- `E1` – echo on (so I see what I'm typing).
- `V1` – verbose result codes.
- `Q0` – result codes enabled.
- `X4` – full result codes including BUSY and NO DIALTONE.
- `S0=0` – do not auto-answer.
- `S7=60` – wait 60 seconds for carrier.
- `S11=55` – tone duration 55 ms (faster dialling).
- `&C1` – DCD follows carrier (terminal program detects hang-up).
- `&D2` – modem hangs up if DTR drops.
- `&K3` – hardware (RTS/CTS) flow control.
- `\N3` – auto-reliable (MNP4 or V.42 with normal fallback).
- `%C1` – MNP5 compression on.

BBS sysops collected these init strings for every modem they
encountered; the same string would break some modems and unlock
high-speed operation on others. The classic "BBS modem
compatibility list" pages cataloged them.

## Dial string modifiers

Within `ATD<string>`:

| Modifier | Meaning |
| --- | --- |
| `T` | Tone dial mode for what follows |
| `P` | Pulse dial mode |
| `,` | Pause (S8 seconds, default 2) |
| `W` | Wait for second dial tone |
| `@` | Wait for 5 seconds of silence |
| `!` | Hook flash (drop carrier for 0.5 s) |
| `;` | Return to command mode after dialling completes |
| `R` | Reverse: caller becomes answerer |
| `S=<n>` | Dial stored phone number n |

Example: `ATDT9,,1714555-1234` – tone dial 9 (outside line on a PBX),
pause twice (4 seconds), then the phone number.

## Why this matters now

Modem-protocol awareness is largely a curiosity in 2025, but every
piece of "modem-aware" software in the BBS era – terminal programs,
mailer queue managers, BBS front-ends, ZModem upload daemons –
ran on this command set. Reading the Hayes AT spec is the only way
to understand why scripts of the era look the way they do, what
"ATZ" does, why init strings are so long, and what a "guard time"
is.

V.250 remains current; modern cellular modems, satellite modems,
and any "AT command" embedded radio device speak a superset of the
Hayes commands described above.
