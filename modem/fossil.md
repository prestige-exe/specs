# FOSSIL – Fido/Opus/SEAdog Standard Interface Layer

by Vince Perriello, Bob Hartman, and Ray Gwinn
The FOSSIL concept originated in 1985 to give Fido, Opus, and SEAdog
mailers a single serial-port API that worked across the incompatible
COM port arrangements of XT, AT, PS/2, and clone hardware. The
specification was formalised in FidoNet Standards Committee documents
FSC-0049 (revision 5, 1989) and later FSC-0079 (revision 6 draft).
Two implementations dominated: X00 by Raymond L. Gwinn (released
1986, maintained through 1997) and BNU by David Nugent (1989 onward).
Smaller implementations include OpusComm, RBcomm, and the Opus mailer
itself which embedded a FOSSIL.

A FOSSIL is a TSR (terminate-and-stay-resident) DOS driver that hooks
software interrupt 14h. DOS's own INT 14h serial services were
limited to 9600 bps, polled rather than interrupt-driven, and unaware
of any UART beyond a 16450. A FOSSIL replaces INT 14h with a richer,
interrupt-driven, buffered, properly-flow-controlled API. Every BBS
package (Opus, RemoteAccess, QuickBBS, Maximus, RBBS, TBBS), every
mailer (BinkleyTerm, FrontDoor, InterMail, D'Bridge), every terminal
program that supported it (Telix with the SALT script command
`fossil`, Telemate, Qmodem-Pro), and every file-transfer engine
(DSZ, ZModem-8K) used the FOSSIL API.

## Activation

A program activates the FOSSIL by calling function 04h (Initialize
Driver) with AH=04h, DX=port number, BX=0x4F50 ("OP" in ASCII, the
signature for "we know this is FOSSIL"). If the FOSSIL is present
and the call succeeds, AX returns 0x1954 (the FOSSIL magic number)
and BL returns the maximum function code supported.

Deinitialisation is function 05h. Programs that crash and leave the
FOSSIL initialised generally cause no harm; reinitialising is
idempotent.

## Function reference

All functions are invoked via INT 14h with the function code in AH.
The port number (0 = COM1, 1 = COM2, etc.) goes in DX. The FOSSIL
spec defines functions 00h through 1Eh; not every implementation
supports the full set.

### AH = 00h – Set Baud Rate

```
AH = 00h
AL = encoded baud rate and line parameters
DX = port
```

AL encodes baud rate in bits 7-5 and line parameters (parity, stop
bits, data bits) in bits 4-0. The encoding matches the IBM ROM-BIOS
INT 14h legacy:

| Bits 7-5 | Baud rate |
| --- | --- |
| 000 | 110 |
| 001 | 150 |
| 010 | 300 |
| 011 | 600 |
| 100 | 1200 |
| 101 | 2400 |
| 110 | 4800 |
| 111 | 9600 |

Higher rates (19200, 38400, 57600, 115200) use function 1Eh.

Returns: AX = port status word.

### AH = 01h – Send Character (with wait)

```
AH = 01h
AL = character to send
DX = port
```

Adds the character to the transmit buffer. If the buffer is full,
the function waits until space becomes available (subject to the
flow-control state). Returns AX = port status.

### AH = 02h – Receive Character (with wait)

```
AH = 02h
DX = port
```

Returns the next character from the receive buffer in AL. If the
buffer is empty, waits until a character arrives. AH on return holds
flags (bit 7 = error). AX may instead return an error code if the
wait is interrupted.

### AH = 03h – Request Status

```
AH = 03h
DX = port
```

Returns the port status word in AX:

| Bit | Set when |
| --- | --- |
| 15 | Timeout |
| 14 | Transmit shift register empty |
| 13 | Transmit holding register empty (room to send) |
| 12 | Break detected |
| 11 | Framing error |
| 10 | Parity error |
| 9 | Overrun error |
| 8 | Receive data ready (room to read) |
| 7 | Carrier detect (DCD) |
| 6 | Ring indicator |
| 5 | Data set ready (DSR) |
| 4 | Clear to send (CTS) |
| 0-3 | Implementation-specific |

Bits 8 and 13 are the ones BBS code checks in its main loop.

### AH = 04h – Initialize Driver

```
AH = 04h
BX = 0x4F50 ('OP')
DX = port
```

Activates the FOSSIL for the named port. The driver allocates buffers,
hooks the UART's IRQ, and configures the 8250/16450/16550 for
interrupt-driven operation. Returns AX = 0x1954 if successful;
BL = highest supported function code (typically 0x1B for X00 and
BNU).

### AH = 05h – Deinitialize Driver

```
AH = 05h
DX = port
```

Releases the port. The driver unhooks the IRQ and restores the
prior INT 14h handler if no other port is still active.

### AH = 06h – Raise/Lower DTR

```
AH = 06h
AL = 0 (drop DTR) or 1 (raise DTR)
DX = port
```

Used by mailers to hang up by dropping DTR (when `&D2` is configured
on the modem).

### AH = 07h – Return Timer Tick Parameters

```
AH = 07h
```

Returns timer characteristics in registers:

- AL = ticks per second (commonly 18)
- AH = approximate ms per tick (commonly 55)
- DX = number of timer interrupts since FOSSIL init (low word)
- CX = number of timer interrupts since FOSSIL init (high word)

Used by terminal programs to compute timeouts without dragging in
DOS time-of-day routines.

### AH = 08h – Flush Output Buffer

```
AH = 08h
DX = port
```

Blocks until the transmit buffer (and the UART's holding and shift
registers) are empty. Used before disconnect or before mode changes.

### AH = 09h – Purge Output Buffer

```
AH = 09h
DX = port
```

Discards any pending output without sending it. Used to abort a
half-sent command after the remote has hung up.

### AH = 0Ah – Purge Input Buffer

```
AH = 0Ah
DX = port
```

Discards any pending input. Used to clear noise after a connect
before starting protocol negotiation.

### AH = 0Bh – Send Character (no wait)

```
AH = 0Bh
AL = character
DX = port
```

Attempts to add a character to the transmit buffer. Returns AX =
0x0001 if accepted, AX = 0x0000 if the buffer is full and the
character was discarded. Polling alternative to function 01h.

### AH = 0Ch – Non-destructive Read (peek)

```
AH = 0Ch
DX = port
```

Returns the next available character in AL without removing it from
the buffer. AH = 0xFF (or AX = 0xFFFF) if the buffer is empty.

### AH = 0Dh – Read Keyboard (no wait)

```
AH = 0Dh
```

Returns the next character from the local keyboard in AX. AX =
0xFFFF if no key is pending. Equivalent to INT 16h AH=01h with
buffer pop, but exposed through the FOSSIL so an application that
already calls INT 14h does not need a separate INT 16h path.

### AH = 0Eh – Read Keyboard (with wait)

```
AH = 0Eh
```

Same as 0Dh but blocks until a key is available. The FOSSIL pumps
the serial port in the wait loop so background data continues to
flow.

### AH = 0Fh – Enable/Disable Flow Control

```
AH = 0Fh
AL = flow-control flags
DX = port
```

AL bits:

| Bit | Meaning |
| --- | --- |
| 0 | Transmit XON/XOFF |
| 1 | CTS/RTS handshake |
| 2 | Receive XON/XOFF |
| 3 | Reserved (must be 0) |

A typical setting for BBS use is AL = 0x02 (hardware flow control
only). File-transfer protocols disable software flow control to
allow 0x11 and 0x13 in the binary stream.

### AH = 10h – Control Ctrl-C / Ctrl-K Checking and Transmitter

```
AH = 10h
AL = bit flags
DX = port
```

AL bits:

- Bit 0: Enable (1) or disable (0) Ctrl-C / Ctrl-K checking on input
  (when enabled, the FOSSIL aborts the calling task on Ctrl-Break).
- Bit 1: Disable (1) or enable (0) the transmitter.

### AH = 11h – Set Cursor Position

```
AH = 11h
DH = row (0-based)
DL = column (0-based)
```

Sets the cursor via the FOSSIL's video output path. The FOSSIL
provides ANSI emulation on the local console so a sysop's status
screen looks identical to the caller's view.

### AH = 12h – Read Cursor Position

```
AH = 12h
```

Returns row in DH, column in DL.

### AH = 13h – Single-character ANSI screen write

```
AH = 13h
AL = character
```

Writes the character to the local console, interpreting ANSI escape
sequences (ESC[ codes) along the way. The FOSSIL maintains its own
ANSI state machine so the application does not need ANSI.SYS loaded.

### AH = 14h – Enable/Disable Watchdog

```
AH = 14h
AL = 0 (disable) or 1 (enable)
DX = port
```

Watchdog mode: if carrier (DCD) drops, the FOSSIL reboots the system.
Used in unattended BBS operation so a wedged BBS task does not tie
up the line forever. The companion function is 15h.

### AH = 15h – Cold Reboot via Watchdog

```
AH = 15h
```

Triggers an immediate cold reboot. The implementation jumps to the
ROM-BIOS reset vector at F000:FFF0. Used after detecting a fatal
condition.

### AH = 16h – Install Application Function (BREAK handler)

```
AH = 16h
ES:DX = far pointer to handler
DX  (low word) = port
```

Installs an application-supplied function that the FOSSIL will call
on certain events (Ctrl-Break, line errors, carrier loss). The exact
signature is implementation-specific; X00 and BNU agree on a near
call with AX containing the event code.

### AH = 17h – Uninstall Application Function

```
AH = 17h
```

Removes the handler installed by 16h.

### AH = 18h – Boot the system

Alias on some implementations for 15h (cold reboot via watchdog).
Other implementations reserve 18h for "Get Driver Info Block" which
returns a far pointer to a structure describing the driver name,
version, and capabilities.

### AH = 19h – Install Application Function (replacement for 16h)

A second slot for installing a callback, used by some applications
that need both an event handler and a polled tick handler.

### AH = 1Ah – Uninstall Application Function (slot 2)

Companion to 19h.

### AH = 1Bh – Get Driver Information Block

```
AH = 1Bh
ES:DI = pointer to receive-buffer
CX = size of receive buffer
DX = port
```

Returns a structured information block describing the driver:

| Offset | Size | Field |
| --- | --- | --- |
| 0 | 2 | Structure size |
| 2 | 1 | FOSSIL spec revision (5 for FSC-0049) |
| 3 | 1 | Driver revision |
| 4 | 4 | Far pointer to driver ID string (ASCIIZ, ending with 0x00) |
| 8 | 2 | Size of input buffer (bytes) |
| 10 | 2 | Bytes free in input buffer |
| 12 | 2 | Size of output buffer (bytes) |
| 14 | 2 | Bytes free in output buffer |
| 16 | 1 | Screen width |
| 17 | 1 | Screen height |
| 18 | 1 | Baud-rate code currently in effect |

The driver ID string is how applications detect "this is X00" versus
"this is BNU" so they can issue driver-specific commands. X00's
string starts with "X00"; BNU's starts with "BNU".

### AH = 1Eh – Set/Get Extended Baud Rate

```
AH = 1Eh
AL = 0 (set) or 1 (get)
BX = baud rate code (when setting)
DX = port
```

Codes when setting:

| BX | Baud rate |
| --- | --- |
| 0 | 19200 |
| 1 | 38400 |
| 2 | 57600 |
| 3 | 115200 |
| 4 | 230400 (driver- and UART-dependent) |

Function 00h's three-bit baud code cannot express anything above
9600. Function 1Eh is the standard way to drive a locked-rate BBS
modem (typically 38400 or 57600 bps DTE) on a 16550 UART.

## Buffer sizing

Both X00 and BNU support configurable input and output buffers via
command-line switches at TSR load time. Typical sysop configuration:

```
X00 E B,1,38400 T=2048 R=2048
```

`B,1,38400` opens COM2 at 38400 bps; `T=2048` sets a 2048-byte
transmit buffer; `R=2048` sets a 2048-byte receive buffer. Generous
buffers absorb the latency between the UART IRQ and the BBS task
servicing function 02h, which is important on multitasking
environments like DESQview where the BBS task might be preempted
for many milliseconds.

## UART support

Both X00 and BNU detect and use the 16550A FIFO when present. The
FIFO trigger level is configurable (commonly 8 or 14 bytes) and the
driver reads or writes whole FIFOs per IRQ. Without a 16550 (an 8250
or 16450), the IRQ rate is one per byte and locked DTE rates above
19200 bps are unreliable. The 16550A was the canonical "BBS UART"
through the early 1990s.

## Multi-port and DESQview considerations

X00 supports up to 8 ports; BNU supports up to 4 (or 8 in later
versions). Each port has its own buffers and its own IRQ hook. Both
drivers are aware of DESQview and the standard Microsoft / Quarterdeck
multi-tasking APIs and will yield CPU when waiting on buffer space
rather than spinning. The `T` (timer) parameter on X00 controls how
often the driver pumps the keyboard for the application's benefit.

## Programs that depend on the FOSSIL

- **Mailers**: BinkleyTerm, FrontDoor, InterMail, D'Bridge, T-Mail.
- **BBS packages**: Opus-CBCS, RemoteAccess, QuickBBS, Maximus, TAG,
  Wildcat (optional), TriBBS.
- **External protocols**: DSZ (ZModem), HS/Link, BiModem, Jmodem.
- **Terminal programs**: Telix (when scripted), Telemate, Terminate.
- **Utilities**: T-Port, Doorway (DOS-redirector for opening DOS apps
  to remote callers).

Doorway is worth a special note: it sits between a BBS and a DOS
application, redirecting all INT 10h (video) calls to the FOSSIL's
INT 14h output and all INT 16h (keyboard) reads to FOSSIL input.
This is how BBSes ran arbitrary DOS programs as door games: the door
program thinks it is on a local console; Doorway translates the
local console traffic to and from the FOSSIL.

## References

- FSC-0049, "FOSSIL Driver Specification, Level 5", Vince Perriello
  and Bob Hartman, FidoNet Technical Standards Committee, 1989.
- FSC-0079, "FOSSIL Driver Specification, Level 6 Proposed", 1991.
- Ray L. Gwinn, "X00 FOSSIL Driver Documentation", distributed with
  X00 versions 1.00 through 1.53.
- David Nugent, "BNU FOSSIL Driver Documentation", distributed with
  BNU versions 1.60 through 1.70.
- Opus-CBCS documentation, in particular the chapters on COM port
  configuration which document the early FOSSIL conventions.
