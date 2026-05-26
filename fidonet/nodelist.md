# Nodelist – FidoNet Nodelist text format

Originally specified by Ben Baker and Randy Bush as FidoNet Technical
Standard FTS-5000 (text format). Flag conventions formalised as
FTS-5001. Distribution (NodeDiff) defined in FTS-0005. The current
revision in use is FTS-5001 series, latest revision FTS-5001.006.

The Nodelist is the weekly-published, world-wide registry of every
FidoNet node: who they are, where they are, what number to dial (or
what host to BinkP), and which capabilities they advertise. Every
mailer builds its routing index from the Nodelist; without it, an
address like `2:280/1029` cannot be resolved to a phone number or IP
host. The Nodelist is plain ASCII (CP437 in practice), with one node
per line, and is updated weekly via the NodeDiff mechanism rather than
being redistributed in full.

## File-level structure

A Nodelist is a single text file with CR LF line endings, named
`NODELIST.nnn` where `nnn` is the three-digit day-of-year of
publication (`001` through `366`). The file is distributed inside an
archive named `NODELIST.Pnn` (with `P` indicating compression: `A` =
ARC, `Z` = ZIP, `J` = ARJ, `L` = LZH, `R` = RAR), where the two-digit
`nn` is the week number.

The first line is a comment containing the publication identity and a
trailing CRC:

```
;A FidoNet Nodelist for Friday, July 3, 1987 -- Day number 184 : 15943
```

The five-digit decimal number at the end is the 16-bit CRC of the
entire file from the first character of the second line through the
EOF, including the CR LF on every line but excluding the file's
trailing EOF byte. The polynomial is the same one ZModem and XModem-CRC
use: x^16 + x^12 + x^5 + 1.

Lines beginning with `;` are comments and are ignored by parsers
(except for the first line, which holds the CRC).

## Node-entry lines

Each non-comment line is eight comma-separated fields:

```
<Keyword>,<Number>,<Name>,<Location>,<Sysop>,<Phone>,<Baud>,<Flags>
```

Spaces inside fields are replaced with underscores. Commas are not
permitted in any field value. The Flags field is itself a list of
comma-separated tokens.

### Keyword field

| Keyword | Meaning |
| --- | --- |
| (empty) | A normal node entry. `Number` is the node number within the current `Host` (or `Region`). |
| `Zone` | Begins a zone definition. `Number` is the zone number. All subsequent lines belong to this zone until the next `Zone`. |
| `Region` | A geographic region within the current zone. `Number` is the region number. |
| `Host` | A net within the current region or zone. `Number` is the net number. All subsequent normal nodes are net members until the next `Host`/`Region`/`Zone`. |
| `Hub` | A routing sub-unit inside a net. `Number` is the hub's own node number, and the entry is duplicated as both a hub marker and as a normal node line. |
| `Pvt` | A node whose phone number is private; `Phone` is `-Unpublished-`. Mail still routes through the host. |
| `Hold` | Temporarily down. Mail accumulates at the host or coordinator until the node returns. Used for sysops on vacation, IP-only nodes that are between providers, etc. |
| `Down` | Not operational. Mail must not be sent. A `Down` flag must not persist for more than two consecutive weeks; after that the entry is removed. |

The hierarchical numbering is implicit: a node's full address is
`<current Zone>:<current Host>/<Number>` (with `<current Region>`
substituting for `<current Host>` when the node sits directly under a
region).

### Numeric fields

`Number` is a decimal integer 1..32767. `Baud` is the maximum line
speed: 300, 1200, 2400, 9600, 19200, 38400, or higher in a roughly
doubling sequence. Many ancient parsers cap at 9600, which is why the
`MAX` flag and the explicit modem-protocol flags carry the real
capability.

### Text fields

`Name` is the system name. `Location` is conventionally `City_State`
or `City_Country`. `Sysop` is the operator's name, underscores for
spaces. `Phone` is a country-area-exchange-number format with hyphen
separators: `1-213-555-1234`, `31-20-5551234`, or `-Unpublished-` for
`Pvt` nodes.

## Flags (FTS-5001)

The eighth field onward is a comma-separated list of flags. Flags
group into categories.

### Modem connect-protocol flags

| Flag | Meaning |
| --- | --- |
| `V21` | CCITT V.21 (300 bps) |
| `V22` | CCITT V.22 (1200 bps) |
| `V29` | CCITT V.29 (9600 half-duplex) |
| `V32` | CCITT V.32 (9600 bps) |
| `V32B` | V.32bis (14400 bps) |
| `V33` | V.33 (14400 4-wire leased) |
| `V34` | V.34 (28800/33600 bps) |
| `V90C` | V.90 client side |
| `V90S` | V.90 server side |
| `H96` | Hayes V.9600 |
| `HST` | US Robotics HST (9600 or 14400 / 16800 / 21600) |
| `H14` | 14400 HST |
| `H16` | 16800 HST |
| `MAX` | Microcom AX series |
| `PEP` | Telebit PEP |
| `CSP` | Compucom Speedmodem |
| `ZYX` | ZyXEL 16800/19200 |
| `Z19` | ZyXEL 19200 |
| `VFC` | V.Fast Class |

Without a modem flag, parsers assume Bell 212A for 1200 bps systems
and CCITT V.22bis for 2400 bps systems.

### Error correction and compression flags

| Flag | Meaning |
| --- | --- |
| `MNP` | MNP error correction (any class) |
| `V42` | V.42 error correction |
| `V42B` | V.42bis compression |
| `MN` | No MNP / no V.42 |

### Operating condition flags

| Flag | Meaning |
| --- | --- |
| `CM` | Continuous Mail. Reachable for netmail outside Zone Mail Hour. |
| `MO` | Mail Only. No human callers; mailer answers all calls. |
| `LO` | Listed Only. Outgoing-only, will not answer. |

### File-request flags

| Flag | Meaning |
| --- | --- |
| `XA` | Bark and WaZOO file requests |
| `XB` | Bark only |
| `XP` | WaZOO only |
| `XC` | Bark, WaZOO, and EMSI file requests |
| `XR` | WaZOO file requests only on T-mail |
| `XW` | WaZOO only, no Bark |
| `XX` | Will not honour file requests |

### Internet / mailer flags (FTS-5001 modern additions)

| Flag | Meaning |
| --- | --- |
| `IBN[:port]` | BinkP over TCP (FTS-1026). Optional port, default 24554. |
| `IFC[:port]` | RAW / ifcico session over TCP. |
| `ITN[:port]` | Telnet to BBS (port 23 default). |
| `IVM[:port]` | Vmodem over TCP. |
| `IFT[:port]` | FTP file inbound. |
| `IEM:<addr>` | Email gateway address. |
| `IMI:<addr>` | Email-encapsulated FidoNet (MIME). |
| `INA:<host>` | Internet host name (DNS) for the above protocols. |
| `INO4` | No IPv4 |
| `IP` | Generic "has an IP address" (pre-IBN era). |

### Mail-period flags

| Flag | Meaning |
| --- | --- |
| `#01`..`#23` | Specific hour of additional mail period (UTC) |
| `Txx` | Trxx hour-pair availability |

### Gateway and miscellaneous

| Flag | Meaning |
| --- | --- |
| `Gxx` | Gateway to domain `xx` (e.g. `Guucp`, `Gfido`) |
| `Uxx` | User-defined |
| `ENC` | Encrypted-session-capable |
| `NPK` | Will not accept packed mail |
| `NEC` | Will not accept encrypted mail |
| `NLO` | No login required for file requests |
| `K12`, `K39`, `K56` | ISDN at various speeds |

Any string the parser does not recognise must be preserved (a future-
proof rule), and the line should still be considered valid.

## NodeDiff

A full Nodelist is on the order of one to three megabytes uncompressed
and changes only modestly week to week. The weekly publication is
actually `NODEDIFF.nnn` (the day-of-year of the new Nodelist), a diff
that lists only added, changed, and removed entries against the
previous week. A NodeDiff processor (`XlatList`, `EditNL`, modern
sbbsecho's NodeDiff handler) applies the diff to last week's
`NODELIST.nnn` to produce this week's. The diff itself is line-based
with simple `Add`, `Delete`, and `Change` markers; format details are
in FTS-5001.

The NodeDiff archive (`NODEDIFF.Pnn`) is distributed via the file echo
named `NODELIST`, the same way any other FidoNet file moves (see
[[tic]]).

## How mailers use it

Every mailer builds an internal binary index keyed by full FidoNet
address. The index points back to the source line in the Nodelist so
that the mailer can read the flags on demand. When an outbound message
needs to be routed, the mailer:

1. Looks up the destination address.
2. Selects a transport based on the destination's flags: an `IBN` flag
   pulls the BinkP path; an `XW`/`XP`/`HST`/`V34` combination implies
   a modem dial.
3. Reads the `Phone` or `INA:` value as the connection target.
4. Checks `CM` to decide whether to call now or queue until Zone Mail
   Hour.

The index file is typically a hash of address-to-offset entries; tools
like XlatList, Parselst, FastLst, and BinkD's own indexer all produce
mailer-specific binary forms (`NODEX.DAT`, `NODE.NDX`, `nodelist.dat`)
from the canonical text file. The text Nodelist is the authoritative
source; binary indexes are derived and rebuilt every week.

## References

- FTS-5000 / FTS-5001: The text Nodelist file format and flags. Latest
  revision FTS-5001.006, ftsc.org.
- FTS-0005: The Distribution Nodelist (originally Ben Baker, Randy
  Bush, and others).
- FTS-1026 / FTS-1027: BinkP, the protocol behind the `IBN` flag.
- FidoNet Policy 4.07, sections on node listing, Hold, and Down.
- XlatList, EditNL, Parselst, FastLst: long-running Nodelist
  processing utilities.
